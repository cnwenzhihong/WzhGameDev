---
created: 2025-11-22T21:19:31+08:00
modified: 2026-01-12T00:28:59+08:00
feature: thumbnails/external/28bffd2484a26253ebd451df99a55ab2.png
thumbnail: thumbnails/resized/82343318e33e845cd5fd43bc6ef2f917_b89e22fb.jpg
tags:
  - 3C
  - FPS
  - blog
dg-publish: "true"
dg-home: "true"
cssclasses:
  - list-cards
title: FPS中的武器摇晃
slug: fpszhong-de-wu-qi-yao-huang
cover: ""
categories: []
halo:
  site: http://192.168.0.107:28090
  name: 8a6707a9-8c42-4ee8-a2e0-d247183fe9c3
  publish: false
date: 2025-11-22T13:19:29.880Z
lastmod: 2026-01-17T12:43:42.619Z
---
1. 加速度方向不一致
2. 一般单次滑动最大加速度
3. 停下：速度方向不会改变
4. 左右变向时：速度方向改变
5. 减速滑动时：缓慢归0

我设想了一个思路：\
当加速度方向和速度方向不一致时判断停止\
如果速度大于40度每秒，则累加DeceTime。否则直接判定NoInput\
之后0.3秒内采用Lerp(NoInput、HasInput)，因为减速时间越长越有可能是单纯的减速\
如果DeceTime>0.3ms，则直接采用HasInput\
速度方向发生改变时充值DeceTime\
持续移动时、加速度会在0附近震荡

# 1. 旋转摇晃Weapon Sway

> \[!note]- 资料\
> [CraftingGunFeel.pdf](https://his.diva-portal.org/smash/get/diva2%3A1971715/FULLTEXT01.pdf) 对武器程序化动画、枪械感觉做出了大量实践指导，极具参考价值，不过较为零散，需要自行整理\
> [使用控制绑定制作武器偏移效果](https://www.bilibili.com/video/BV1C6BEBHEXG):使用control rig实现相同效果的优秀示例

## 1.1. 代码实现

我们可以将其分为：获取转动输入→计算偏移量→表现三个步骤

> 先获取用户转动差值，做各种处理后，用于表现效果

http://125.208.22.164/#/convert?card=D2F9B25D74AF8ACA

### 1.1.1. 一、处理输入量

> 计算转动量/输入量的速度，用于后续处理

#### 1.1 原始数据采样——转动量

类似于TPS的瞄准偏移，但TPS关注角度，而FPS更在意变化量

##### 鼠标输入 ★★★

理论上是最直接简单的获取方式，完全不需要计算\
不容易获得精确数值(要么0要么1)。\
UE愈加复杂的输入系统让实现不再简单。

> \[!example]- 实现\
> 0.1秒间隔，无太大必要优化，并未处于deltaSeconds\
> ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251126063200443.png)

##### 旋转差值 ★★★★★

计算`ControllerRotation`的旋转差值\
可以抹平不同鼠标Dpi的差距，获取差值也简单，需要单独保存上一帧的旋转量\
![|300](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251216011431739.png)

> \[!example]-
>
> ```cpp
> FRotator deltaRot = UKismetMathLibrary::NormalizedDeltaRotator(CurrtRot, ControlLastRot); 
> ControlLastRot = CurrtRot;  
> ```
>
> > Rotator.Normalized:\
> > 防止接近 ±180° 时出现跳变\
> > 消除超过 ±360° 的累积角度\
> > Yaw Roll：\[=180,180]\
> > Yaw：\[-90,90]

#### 1.2 计算二阶数据——速度和加速度

##### 角速度

差值➗时间 即可得到角速度\
`s=d/t`

```cpp
FVector2D rotSpeedFrame(deltaRot.Yaw / DeltaSeconds, deltaRot.Pitch / DeltaSeconds);
```

##### 加速度

`a=s/t`

```cpp
FVector2D rotAccel((RotSpeedFiltered.X - PrevRotSpeedFiltered.X) / DeltaSeconds,  
                   (RotSpeedFiltered.Y - PrevRotSpeedFiltered.Y) / DeltaSeconds);  
PrevRotSpeedFiltered.X = RotSpeedFiltered.X;  
PrevRotSpeedFiltered.Y = RotSpeedFiltered.Y;
```

#### 1.3 输入数据处理

> 主要对算出的Rot speed多次处理

##### 低通滤波

> 平滑变化速率较大的情况

适用场景：

* 大幅度摆动时，防止短时间参数跳变引起Target跳变
* Interp 平滑输入，会看起来更舒缓，但变化减慢。

> \[!example]- 代码
>
> ```cpp
> if (bUseInputLowpass)
> {  
>     float alpha = FMath::Clamp(DeltaSeconds * 10.f, 0.f, 1.f);  
>     RotSpeedFiltered.X = FMath::Lerp(RotSpeedFiltered.X, rotSpeedFrame.X, alpha);  
>     RotSpeedFiltered.Y = FMath::Lerp(RotSpeedFiltered.Y, rotSpeedFrame.Y, alpha);  
> }  
> else  
> {  
>     RotSpeedFiltered.X=rotSpeedFrame.X;  
>     RotSpeedFiltered.Y=rotSpeedFrame.Y;  
> }
> ```

> \[!example]- 低通滤波效果对比\
> 曲线对比：\
> ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251215013753758.png)\
> 视觉效果：\
> Normal:\ <video src="https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/NoLowpass.mp4" controls muted autoplay loop></video>\
> Lowpass\ <video src="https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/Lowpass.mp4" controls muted autoplay loop></video>

##### 归一化

> 将数值映射到\[-1,1]，方便后续操作

角速度理论上可以无限大，需要限制最大值，顺带将其映射到\[-1,1]，之后便不需要再考虑输入量的实际大小

###### 鼠标输入 ★

归一化受鼠标DPI影响，很难设定一个合理的数值范围，自定义程度弱(仅有最高和最低)，但不是不能做。

###### 旋转差值 ★★★★★

可以规定每秒转动角度的最大值：如3600，在\[-3600,+3600]内很容易控制表现效果。参考最大值：1440°/s

```cpp
//用一个“参考最大转速”fiducialRotSpeed 来归一化
FVector2D tmp_RotSpeed = originRotSpeed / fiducialRotSpeed;
FVector2D rotSpeedNormalize = FVector2D(FMath::Clamp(tmp_RotSpeed.X, -1, 1), FMath::Clamp(tmp_RotSpeed.Y, -1, 1));
```

##### 非线性映射

需求1-更容易瞄准：小幅度转动时提供更小的视觉变化，大幅度转动时再提升。(> 0)\
需求2-更灵敏：小幅度转动时(刚动鼠标即可响应)便能有可见的摆动。(< 0)\
指数函数都能较低成本地实现，也可以改为曲线精准调节\
![|425](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251226065943488.png)

> \[!example]- 示例
>
> ```cpp
> //可以改为曲线
> auto NonLinear = [](float v) { return FMath::Sign(v) * FMath::Pow(FMath::Abs(v), 1.15f); };  
> float nx = NonLinear(rotSpeedNorm.X);  
> float ny = NonLinear(rotSpeedNorm.Y);
> ```

### 1.1.2. 二、计算偏移量

我们需要得到WeaponSway的关键结果：`SwayOffset`和`SwayRot`

#### 2.1 方案一：计算Sway Target

通过上面获取的输入量(偏移/速度/加速度)，处理得到想要的偏移和旋转目标值。

> \[!important]+\
> [3D 软件坐标系类型与转换](https://zhuanlan.zhihu.com/p/641172693)\
> 鼠标输入与UE坐标轴映射关系——以UE为准：
>
> * X → Y
> * Y → Z\
>   ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251226164442511.png)\
>   欧拉角也采用UE标准\
>   ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251226170200899.png)\
>   武器常见的坐标系\
>   ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251226182802678.png)

> \[!example]- 代码
>
> ```cpp
> swayOffsetTarget = FVector(  
>     ControllerRotSpeed_Rate.X * SwayOffsetMulti.X * -1.f,  
>     ControllerRotSpeed_Rate.X * SwayOffsetMulti.Y,  
>     ControllerRotSpeed_Rate.Y * SwayOffsetMulti.Z  
> );  
> //rotator构造函数顺序和蓝图显示顺序不一样，赋值能防止写参数时顺序混乱  
> swayRotTarget.Roll=ControllerRotSpeed_Rate.X * SwayRotMulti.Roll;  
> swayRotTarget.Pitch=ControllerRotSpeed_Rate.Y * SwayRotMulti.Pitch * -1.f;  
> swayRotTarget.Yaw=ControllerRotSpeed_Rate.X * SwayRotMulti.Yaw;
> ```

##### Tips: 左右旋转时加上Pitch值

> \[!tip] 左旋转时，Pitch向下压是一种优秀的效果\ <video src="https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/WeaponSway_PitchAdd.mp4" controls muted autoplay loop></video>\
> 理论上我们应将左右、上下旋转做独立的Vector和Rotator映射，而不是直接指定轴。但偷了懒，且目前的参数够用

上文代码中鼠标X移动控制Yaw和Roll旋转，但部分游戏中添加Pitch旋转能进一步增强效果，对其进行补充

> \[!example]- 代码
>
> ```cpp
> void UWeaponRotationSwayFragment::MouseXRotModify(FRotator& swayRotTarget) const  
> {  
>     //Yaw叠加Roll  
>     auto pitchAdditionMulti= ControllerRotSpeedNormalized.X<0 ? PitchAdditionMultiWithYaw.X:PitchAdditionMultiWithYaw.Y;  
>     float pitch_Addition=ControllerRotSpeedNormalized.X  * pitchAdditionMulti * FMath::Pow(1-abs(ControllerRotSpeedNormalized.Y),PitchAdditionPower) * -1;  
>     swayRotTarget.Pitch+=pitch_Addition;  
> }
> ```

#### 2.2 方案一：对Sway Target插值

> \[!important] 武器摇晃手感的核心

##### 常用函数

###### InterpTo：简单插值

简单，效果也很不错，比较稳重，视觉变化小\
参数配置轻松

> \[!example]- InterpTo代码
>
> ```cpp
> if (SwayOffset!=swayOffsetTarget)  
> {  
>     SwayOffset = UKismetMathLibrary::VInterpTo( SwayOffset,swayOffsetTarget,DeltaSeconds,bHasInput?InterpSpeed_Pos_Input:InterpSpeed_Pos_NoInput);  
> }  
> if (SwayRot!=swayRotTarget)  
> {  
>      SwayRot = UKismetMathLibrary::RInterpTo( SwayRot,swayRotTarget,DeltaSeconds,bHasInput?InterpSpeed_Rot_Input:InterpSpeed_Rot_NoInput);  
> }
> ```

###### SpringInterpTo：模拟弹簧

更灵动、反馈感更强，视觉变化快\
参数配置麻烦

> \[!example]- SpringInterpTo代码
>
> ```cpp
> //position spring interp  
> SwayOffset = UKismetMathLibrary::VectorSpringInterp(SwayOffset, swayOffsetTarget, PosSpringState,  
>                                                     Params.Pos_Stiffness, Params.Pos_Damping, DeltaSeconds,  
>                                                     Params.Pos_Mass,Params.TargetVelocityAmount);  
> // clamp  
> if (SwayOffset.Size() > Params.MaxOffset)  
> {  
>     SwayOffset = SwayOffset.GetSafeNormal() * Params.MaxOffset;  
> }  
>   
> //rotation spring interp  
> SwayRot = RotatorSpringInterp(SwayRot, swayRotTarget, RotSpringState, Params.Rot_Stiffness, Params.Rot_Damping,  
>                               DeltaSeconds, Params.Rot_Mass,Params.MaxRot);
>                             
> ```
>
> RotatorSpringInterp实现
>
> ```cpp
> FRotator UPawnComp_FPSRotWeaponSway::RotatorSpringInterp(const FRotator& Current, const FRotator& Target,  
>                                                          FRotatorSpringState& State, float Stiffness, float Damping,  
>                                                          float DeltaTime, float Mass,FRotator MaxRot)  
> {  
>     DeltaTime=DeltaTime>RotMaxDeltaSeconds?RotMaxDeltaSeconds:DeltaTime;  
>     // Convert to vector for simple integration (Pitch, Yaw, Roll)  
>     FVector cur(Current.Pitch, Current.Yaw, Current.Roll);  
>     FVector targ(Target.Pitch, Target.Yaw, Target.Roll);  
>   
>   
>     // Semi-implicit Euler integration  
>     FVector acc = (targ - cur) * Stiffness - State.Velocity * Damping;  
>     acc = acc / FMath::Max(Mass, KINDA_SMALL_NUMBER);  
>     State.Velocity += acc * DeltaTime;  
>     FVector next = cur + State.Velocity * DeltaTime;  
>   
>   
>     FRotator nextRot = FRotator(next.X, next.Y, next.Z);  
>     if (!MaxRot.IsZero())  
>     {       nextRot.Pitch = FMath::Clamp(nextRot.Pitch, -MaxRot.Pitch, MaxRot.Pitch);  
>        nextRot.Yaw = FMath::Clamp(nextRot.Yaw, -MaxRot.Yaw, MaxRot.Yaw);  
>        nextRot.Roll = FMath::Clamp(nextRot.Roll, -MaxRot.Roll, MaxRot.Roll);  
>     }    return nextRot;  
> }
> ```
>
> FRotatorSpringState
>
> ```cpp
> USTRUCT(BlueprintType)  
> struct FRotatorSpringState  
> {  
>     GENERATED_BODY()  
>     UPROPERTY()  
>     FVector Velocity = FVector::ZeroVector; // pitch, yaw, roll velocity  
> };
> ```

> \[!note]+ 参数\
> 可视化网站：\
> target velocity amount: 到达目标的速度，0-1:紧贴目标\
> stiffness: 越高越会跟上目标\
> Critical Damping Factor:0-1 多有弹性，0:像橡皮筋\
> k: stiffness 刚度值 控制加速度：越高加速越快，慢慢随距离衰减\
> d: damping factor 阻尼 控制速度衰减(0-1)，越低越减速

> \[!note]- 资料\
> [参数介绍](https://www.youtube.com/watch?v=8-jxARLG2Ik)\
> [Spring-based Animation and Quaternions](https://toqoz.fyi/springs.html?utm_source=chatgpt.com)\
> [优化实现：对数值弹簧的精确控制](https://www.reddit.com/r/gamedev/comments/31vwyg/game_math_precise_control_over_numeric_springing/?utm_source=chatgpt.com)\
> [Damped Springsa原理与实现](https://www.ryanjuckett.com/damped-springs/)\
> [UE4中的阻尼弹簧平滑](https://zhuanlan.zhihu.com/p/412291354)

#### 方案一：SpringInterp介绍

SpringInterp使当前位置朝目标平滑过渡，并能表现出惯性、超调（如果阻尼不足）或快速回弹（如果阻尼大）。\
SpringInterp比简单的Interp更接近物理，能自然模拟质量、刚度、阻尼的不同组合，产生丰富的“尾巴/回弹”效果。

局限性：\
Weapon Sway 每一帧都在更新Target，这会导致：

* Target 不断变化
* SpringState 一直被重写
* 内部 velocity 失真

用于旋转的弹簧插值\
社区推荐用自己的弹簧求解[Spring-based Animation and Quaternions](https://toqoz.fyi/springs.html?utm_source=chatgpt.com)

#### 2.3 方案二：自定义简易弹簧插值

在SpringInterp中，输入和回正是由函数自行计算的，我们无法精确控制。\
在做并不需要那么有弹性的系统时，完全可以自行用一套力反馈机制，让玩家自己去推Weapon

> 优点：\
> ✅ 有惯性 ✅ 速度过渡更自然 ✅ 有回正 ✅ 有重量\
> 缺点：\
> 无法准确预估Target、只有Rot没有位移(有需求可以加)

**推力**\
通过上面获取的输入量(偏移/速度/加速度)，得到一个持续朝运动方向推的力，而不是一个目标点。

> \[!note]- 示例
>
> ```CPP
> WeaponAngVel.Pitch += -AngularSpeed.Y * SwayRotMulti.Pitch;
> WeaponAngVel.Yaw   += -AngularSpeed.X * SwayRotMulti.Yaw;
> WeaponAngVel.Roll  += -AngularSpeed.X * SwayRotMulti.Roll;
> ```

很明显，如果不加以限制，速度(WeaponAngVel)会不断上升\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260104133021697.png)

**回正力**\
枪自己想回到中间（回正力），越远力越大\
是否有输入：

* 还在甩鼠标  → 枪 **没那么急着回正**
* 停下来 → 枪 **更积极回正**

> \[!note]- 代码
>
> ```CPP
> WeaponAngVel.Pitch += -WeaponRot.Pitch * DeltaSeconds * ForceMulti_HasInput.Pitch * (bHasInput ? 1.f : ForceNoInputRate.Pitch);
> ```

**阻尼**\
让枪慢慢停下来， $f(x)=e^x$  x<0时，f(x)<1\
给Roll的较小的阻尼以便让其有轻微弹性很容易增加质感

> \[!note]- 代码
>
> ```CPP
>  WeaponAngVel *= FMath::Exp(Damp * DeltaSeconds);
> ```

**强制回正**\
场景：如果玩家持续匀速转动，画面很容易保持静止\
可以用噪声缓解，也可以根据输入时间进行缓慢回正，这符合现实直觉

> \[!exaple]- 代码
>
> ```CPP
> if (bHasInput&&Straighten_TotalTime!=0)  
> {  
>     WeaponRot *= FMath::Max(FMath::Pow((Straighten_TotalTime - InputTime) / Straighten_TotalTime, Straighten_NoneLinear), Straighten_MinValue);  
> }
> ```
>
> 假设：\
> Straighten\_TotalTime=2 Straighten\_NoneLinear=0.3 Straighten\_MinValue=0.3\
> 如图：\
> ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260105191120670.png)

#### 2.4 方案二：计算位移

> \[!note]- 代码
>
> ```CPP
> WeaponRot.Pitch += WeaponAngVel.Pitch * DeltaSeconds;
> WeaponRot.Yaw   += WeaponAngVel.Yaw   * DeltaSeconds;
> WeaponRot.Roll  += WeaponAngVel.Roll  * DeltaSeconds;
> ```

### 1.1.3. 后期处理

#### 限幅（MaxOffset / MaxRot）

保护视觉（防止武器飘出画面或产生穿模），并保证在极端输入下效果可控。\
用户灵敏度、鼠标抖动或跳帧可能造成瞬时超大值，限幅能保护不破坏玩家视觉体验。上述情况都很容易出现

应用：\
SpringInterp这种调参困难方案时的\
尽量在运算参数时能预估最大范围，这是最后一道保险。

问题：\
当达到最大值时速度会突然归0，极为生硬，可以用Noise缓解。\
也可以在算到最大值时动态降低速度，但属于本末倒置，我们应专注于插值实现更精准的控制，而不是期待粗暴的最大值限制。

> \[!example]- 代码
>
> ```cpp
> if (SwayOffset.Size() > Params.MaxOffset) SwayOffset = SwayOffset.GetSafeNormal() * Params.MaxOffset;  
> SwayRot.Pitch = FMath::Clamp(SwayRot.Pitch, -Params.MaxRot.Pitch, Params.MaxRot.Pitch);  
> SwayRot.Yaw = FMath::Clamp(SwayRot.Yaw, -Params.MaxRot.Yaw, Params.MaxRot.Yaw);  
> SwayRot.Roll = FMath::Clamp(SwayRot.Roll, -Params.MaxRot.Roll, Params.MaxRot.Roll);
> ```

#### 改变旋转中心

有的文章中提到，Yaw轴旋转可以设置为绕枪托旋转效果会更好\
下图以枪口为轴枢点更方便理解：\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/weaponsway_yawpivot.png)

> \[!example]- 代码
>
> ```cpp
> // 8) Yaw旋转优化  
> FVector PivotLoc=FVector::ZeroVector;  
> // 8.1) 先尝试获取Socket，没有再获取Vector  
> if (!Params.YawPivotSocket.IsNone())  
> {  
>     if (auto weaponMesh= ULyraFPSFunctionLibrary::GetWeaponMesh(GetOwner()))  
>     {       PivotLoc=weaponMesh->GetSocketTransform(Params.YawPivotSocket,RTS_Component).GetLocation();  
>     }}  
> if (PivotLoc==FVector::ZeroVector)  
> {  
>     PivotLoc=Params.YawPivotVector;  
> }  
>     // 8.2) 正式处理  
> if (PivotLoc!=FVector::ZeroVector&&!SwayRot.IsNearlyZero())  
> {  
>     FVector aroundLoc;  
>     FRotator aroundRot;  
>     UWeaponSwayFunctionlibrary::RotateAroundPivot(PivotLoc,FRotator(0,SwayRot.Yaw,0) ,aroundLoc,aroundRot);  
>     SwayOffset.X=aroundLoc.X;  
>     SwayOffset.Y=aroundLoc.Y;  
>     SwayRot.Yaw=aroundRot.Yaw;  
> }
> ```
>
> * 绕某个 Pivot 点做旋转（任意旋转），返回变换后的位置与旋转
> * 原理：T = Pivot + R \* (Pos - Pivot)
>
> ```cpp
> void UWeaponSwayFunctionlibrary::RotateAroundPivot(FVector Pivot, const FRotator& DeltaRot, FVector& OutPosition,  
>     FRotator& OutRotation, const FVector InPosition, const FRotator InRotation)  
> {  
>     // Step 1: 相对 Pivot 的偏移  
>     FVector RelPos = InPosition - Pivot;  
>   
>     // Step 2: 构建旋转 Quat    FQuat QuatRot = DeltaRot.Quaternion();  
>   
>     // Step 3: 应用旋转  
>     FVector RotatedRelPos = QuatRot * RelPos;  
>   
>     // Step 4: 恢复到世界位置  
>     OutPosition = Pivot + RotatedRelPos;  
>   
>     // Step 5: 应用旋转叠加  
>     OutRotation = (FRotationMatrix(InRotation).ToQuat() * QuatRot).Rotator();  
> }
> ```

蓝图使用：\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251218042239133.png)

> \[!example]- 效果\
> 枪托为中心(Pivot=(0,-5,0)) <video src="https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/2025-12-18%2005-16-05.mp4" controls muted autoplay loop></video>\
> 枪头为中心(Pivot=(0,10,0))<video src="https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/2025-12-18%2005-19-39.mp4" controls muted autoplay loop></video>

#### 噪声

如果限制最大值，当武器到达最大偏移值时效果便会生硬(不再运动)，此时加入适当的噪声可以大幅度削减该效果，同时增加适当的随机抖动也可以增加一定效果

暂时在Rot中加入柏林噪声\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20260104233131253.png)

#### Add Pitch

有的情况下存在向\
左旋转时枪身轻微向下的需求，便单独做一个参数控制

> 最好的情况是每个输入轴单独控制完整的Rotator+合并控制Rotator，一共三种控制相加，但配置起来太过麻烦，且还没有如此复杂的需求，暂时搁置。\
> 即便制作也是单独开新类，减少简单需求的输入量

#### HasInput

有无输入时配置不同的参数\
资料：\
[INTERMITTENT CONTROL AS A MODEL OF MOUSE MOVEMENTS](https://arxiv.org/pdf/2103.08558)\
[ControlTheory,DynamicsandContinuousInteraction](https://www.dcs.gla.ac.uk/~rod/publications/Mur18.pdf)

### 1.1.4. 应用函数

#### 基本使用

在Tick中使用\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251218041755810.png)

将计算出的参数用于修改武器IK骨骼即可\
![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251218035612026.png)

## 1.2. 参数配置

### 1.2.1. 优秀的实现案例

#### UE示例

[【自制COD20 ?】开发日志5-绕任意轴旋转枪身](https://www.bilibili.com/video/BV14UfZYaEQn)：左右旋转采用三轴共转能实现极为大开大合的效果\
[使用控制绑定制作武器偏移效果](https://www.bilibili.com/video/BV1C6BEBHEXG)：使用`Control Rig`+`Sprint Interp`快速高效实现非常易于控制的武器旋转摇晃

#### 游戏示例

> \[!example]- cod2——Lead，无延迟，沉重、现实且响应及时\
> [COD2的武器摇摆](https://www.youtube.com/watch?v=kjVSLsz8dCs\&source_ve_path=MTc4NDI0)

> \[!example]- cs2——Lag，少量延迟，停下时立刻追赶上，给人感觉并不拖沓\ <video src="https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/RotSway_CS2.mp4" controls muted autoplay loop></video>

### 1.2.2. 核心思路

#### 第三方资料

> \[!quote]+  [CraftingGunFeel.pdf](https://his.diva-portal.org/smash/get/diva2%3A1971715/FULLTEXT01.pdf) 对武器摇晃的讨论\
> **受访者对武器摇晃的理解**\
> 向前移动时武器的Pitch会略微向下旋转，而向后走则相反。\
> 当角色旋转时，枪械可能跟随相机的旋转滞后，也可能领先。在旋转时，如果领先，武器可能向旋转方向滚动；如果滞后，则可能向相反方向滚动。\
> Cody进一步描述了他对晃动的方法：玩家旋转时武器滞后，然后过冲，最后稳定下来。\
> 在移动稳定时使用弹簧插值
>
> **过冲和滚动Roll的重要性**\
> 当枪的枪托命中肩膀时， 它应该在稳定到预定姿势之前稍微过冲一点。在冲击点之后，应该会有2‑4个快速的小幅度左右滚动，同样遵循过冲原则，然后稳定在中间位置。
>
> **晃动Sway**\
> 从玩家静止时产生的晃动噪音开始。\
> 每个旋转轴上的两层柏林噪声形式的噪声由玩家的生命值和当前武器的控制等级驱动。武器的控制等级。这些元素影响静止晃动的振幅和频率，其中控制性较差的武器晃动振幅更大但频率更低。
>
> 其他类型的晃动，分别由玩家的移动和旋转引起，均由简化的物理驱动驱动，模拟弹簧阻尼器以实现自然的晃动和稳定感。\
> Cody使用弹簧插值来处理武器移动后的回弹和稳定。
>
> 重要的是，Albie的程序化系统允许开发者指定武器在每个轴上操纵的难易程度。Albie指出这是一个强大的特性，因为不同的武器具有不同的重量和质心，并且不同轴上的移动应该以不同方式工作。\
> 例如，武器在偏航轴(Yaw)上旋转时应具有更多阻力，而在滚转轴上的旋转应具有较少阻力、更高频率和更低惯性。武器应更倾向于在滚转轴(Roll)上旋转而非其他轴旋转的想法反复出现，因为Cody、 Anders和Milton都提到利用滚转来实现武器动画的微妙自然移动感。\
> 武器通常在偏航轴(Yaw)上不能运动太多，如果你必须让武器产生 Yaw 摆动，它必须绕枪托（buttstock）旋转，而不是绕枪的中心点或手部位置。\
> pitch / roll 也可以部分受这里影响（取决于你想要的风格）
>
> > **为什么要这么做**：\
> > ◆ 枪在现实中怎么转？\
> > 当玩家身体左右摇动或做非常轻微的 Yaw 旋转时：
> >
> > * 枪托贴在肩膀
> > * 枪托基本是“固定点”，或是偏刚性的支点
> > * 枪管端（muzzle）会动得最大
> > * 武器中部次之
> > * 枪托最小\
> >   📌 这就是现实中的**杠杆**效果。
> >
> > ◆ 如果旋转点错了，会出现什么问题？\
> > 如果你让武器绕 “weapon root” 或 “枪中心” 旋转：
> >
> > * 枪会像“漂浮物”一样左右转
> > * 缺乏重量
> > * 与肩膀没有机械联系
> > * 视觉上非常假（很多 indie FPS 都犯这个错）
> >
> > ◆ 这样做后\
> > ✔ 枪口摆动会被放大\
> > ✔ 枪托稳定，不会乱飞\
> > ✔ 枪看起来像“真枪”，有重量、有支撑点\
> > ✔ 手感立即提升（非常明显）
>
> 在滚转轴上旋转之所以有效，是因为它在不显著影响瞄准方向的情况下增加了不确定感。然而，这在很大程度上取决于特定武器的枢轴点。虽然长武器通常滚动起来感觉良好，但高的武器则不然，因为枪管和瞄准具之间的距离可能会在玩家瞄准时造成不舒服或夸张动作。对此的解决方案很直接——在瞄准时将高武器的枢轴点向上移动。
>
> **视图模型过冲的特性**\
> 是SCP: 5K的一个显著特性。\
> 当玩家转向时，视图模型会超过屏幕中心，并在玩家停止旋转时保持向转向方向偏移。与游戏中的大多数程序化运动相反，这被应用于角色绑定的脊柱。这是为了让旋转发生在网格空间中，意味着它相对于角色的根变换而不是骨骼链中更下方的骨骼。\
> 视图模型过冲不会减少摄像机移动，而是在玩家输入的基础上为视图模型添加额外的移动，并将该移动限制在屏幕的特定区域内。然后根据诸如玩家是否正在瞄准或他们的武器是否有枪托等因素来缩放此效果的振幅。在瞄准射击时， 有托武器如步枪不会出现过冲，而手枪和无托步枪仍然会出现。\
> 手枪尤其因为其长度短、重量轻且缺乏枪托， 容易抖动和旋转，这使得对准其瞄准具变得困难。虽然VR以外的游戏不模拟双焦瞄准，也不要求玩家像在现实世界中那样对准瞄准具，但SCP: 5K的过冲系统仍然能让游戏传达无托武器缺乏稳定性的特点。然而，为了确保玩家仍能命中他们瞄准的位置，摄像机在瞄准射击时会微妙地自动对准手枪后方，同时仍允许视图模型离开屏幕中心。实现这一点的数学计算相当复杂，由于时间关系在访谈中未深入探讨。但他确实指出，一些玩家可能不喜欢这个过冲特性，可能是因为它引发了晕动症，或者因为相关玩家习惯于主要基于十字准星来定位自己，这就是为什么他的团队决定允许玩家减少或关闭此特性。
>
> **美学考量**\
> 如果枪械引导移动，这可能会给玩家一种武器操作者技术娴熟或经验丰富的感觉；而如果武器滞后于摄像机，则可能表明相反的情况。晃动插值的各个方面，例如重新居中所需的时间，可以用来强调武器的尺寸或重量。\
> 一些可用于传达武器特性（如尺寸和重量）的参数包括视图模型过渡到冲刺动画的速度、切换武器的速度、切换到武器后直到可用的时间以及玩家角色瞄准瞄准具的速度。他还引用了后坐力的强度作为可以传达武器威力的一个因素。

#### Input 有无输入对变化速率的影响

##### 平滑速率

一般游戏采用方案，旋转和停下时都采用同一套参数

##### 先慢后快

适用环境：稳重+不拖沓\
特点：鼠标停下后立刻加快插值速度，营造确认感。

> \[!note]- SprintInterp的方案参考(AI)
>
> * CriticalDamping 通常设为 `2 * sqrt(Stiffness * Mass)` 才为临界阻尼，设置不同值，可得到欠阻尼（震荡）或过阻尼（缓慢）效果。建议理解该物理背景。
> * 可以尝试的参数配置：Stiffness 中等偏高（快速响应但不过火）、CriticalDamping 多数接近 1（轻微震荡但不发狂）、Mass 视武器重量或状态而定（重武器 Mass大→摆动慢）。
> * 大 Mass + 大 Stiffness 可能导致高频震荡或振幅异常。
> * 每帧确认 deltaSeconds 非常小的情况下数值稳定性，避免弹簧公式爆炸（如果 stiffness or mass 过大且 delta small）。可做最大 delta clamp。

> \[!note] 增强方案\
> 最好的效果：玩家向右转，武器向左并稍微上/下摆动、同时绕自身轻微旋转
>
> * swayOffsetMulti用 **速度曲线（非线性）** 替代：比如用速度^n 或平滑函数（例如 SmoothStep）以便高转速更强但低转速微弱。
> * 为偏移和旋转分别设计 **不同方向响应**：例如 yaw→横向 + roll，pitch→纵向 + pitch 轻微。

#### Input 非线性映射

为了在不改变整体效果(最大摆动不缩水)的前提下，改善小范围摆动的视觉效果\
采用简单的函数实现：$y=x^n$ ，可以简单分为n>1和n<1\
n > 1：小角度变化较小，偏向让玩家瞄准\
n < 1：大角度变化也大，偏向表现效果

#### Target 映射轴的控制

一般左右旋转控制Yaw轴、上下旋转控制Pitch轴、左右移动控制Roll轴，但左右旋转控制Roll在减少视野变化的前提下效果也不错\
真实武器摆动里 yaw 幅度比 pitch/roll 小，roll（武器绕镜）更能体现质感。

#### Target 超前(Lead)与滞后(Lag)

前者看起来更灵敏(响应强、感觉更爽快)，后者看起来更真实有惯性\
左右超前，**上下追赶**更贴近人的潜意识

##### 示例

> \[!example]- 超前
>
> * UE5 教程1\
>   https://www.youtube.com/watch?v=rZUfkXaVvX4\
>   教程中提到，给模型添加的SpringArm的方案效果并不好，因为无法限制他、并且很晃\
>   ![|350](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-26_07-20-24.gif)
> * UE5 教程2\
>   https://www.youtube.com/watch?v=YahaZR6Umxo\
>   ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-26_07-23-44.gif)
> * UE5 教程3\
>   https://www.youtube.com/watch?v=eYsjiw0fXvU\
>   视频中提到，我们可以反转参数来实现追赶效果。\
>   ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-27_01-33-44.gif)
> * UE 实例1\
>   https://www.youtube.com/watch?v=cpPOA6yKsoo\
>   左右移动时采用roll旋转，鼠标X旋转用较小的Yaw，整体效果非常优秀<video src="https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-27_17-35-54.mp4" controls muted autoplay loop></video>

> \[!example]- 追赶：
>
> * UE Example\
>   [Realistic Weapon Sway In UE5](https://www.youtube.com/watch?v=MMvBRQtjFmQ)\
>   ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-27_01-30-43.gif)

#### Target 锚点(Pivot)

常见的锚点有：围绕角色旋转、围绕左右手旋转、围绕右手偏移、围绕枪托旋转Yaw

##### 绕角色旋转：

很多老游戏中，如虚幻竞技场，一般用Camera Lag就能实现，不需要专门计算。\
优点：容易实现，视野变化大带来较强反馈感，一般搭配追赶效果，体现沉重感。\
问题：无法精确控制效果、效果单一

> \[!example]- UE社区示例\
> https://forums.unrealengine.com/t/weapon-sway-solved\
> [TFE Game - Rotation Lag/Leading](https://www.youtube.com/watch?v=6OFmhQ55iJo)\
> 除了旋转滞后/领先之外，根据滞后/领先值做一些相对位置偏移，效果会好很多。<video src="https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/TFE%20Game%20-%20Rotation%20Lag%E2%A7%B8Leading.mp4" controls muted autoplay loop></video>
>
> ```cpp
>   void APWNWeapon::UpdateLocRot(FVector newLoc, FRotator newRot, float deltaTime) { 
>   FRotator finalRot = newRot; 
>   // Rotation Leading/Lag
>   float lagDiff; 
>   finalRot.Pitch = LagWeaponRotation(newRot.Pitch, LastRotation.Pitch, deltaTime, MaxPitchLag, 1, lagDiff); 
>   finalRot.Yaw = LagWeaponRotation(newRot.Yaw, LastRotation.Yaw, deltaTime, MaxYawLag, 0, lagDiff); finalRot.Roll = newRot.Roll; 
>   // Location offset 
>   float locDiff = lagDiff / MaxYawLag;
>   locDiff *= MaxLocLagX; 
>   ArmMesh->SetRelativeLocation(FVector(ArmMeshOffset.X, ArmMeshOffset.Y + locDiff, ArmMeshOffset.Z), false); 
>   // Set our loc 
>   LastRotation = newRot;
>   this->SetActorLocation(newLoc); this->SetActorRotation(finalRot); 
> ```

##### 围绕手/武器重心旋转

被大量游戏和教程采用\
优点：容易实现，视角变化适中，大都为超前效果，看起来响应很快，应用于双手步枪有非常好的效果。\
局限：不适合重心不处于中心的/较重的枪械

##### 围绕手/武器偏移

https://www.youtube.com/watch?v=u9SuIsd7Dlw

> \[!example]-
>
> * 军团要塞2\
>   [Weapon Sway in Team Fortress 2 Again](https://www.youtube.com/watch?v=4wPAzutIVpc)\
>   ![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-27_16-40-17.gif)\
>   [UFPS - Procedural camera & animation system intro (2012)](https://www.youtube.com/watch?v=Hu0v0kTWH7w)

##### 围绕手/武器旋转+偏移

可以得到最符合直觉(不一定表现最强)的质感：玩家向右猛转→武器轻微绕镜顺转+向左偏移

##### 绕枪托旋转Yaw

把绕 Yaw 的旋转枢轴定位在枪托（贴肩处），保证枪托相对稳定，枪口移动最大，符合现实杠杆效果。\
现实中枪托靠肩为固定支点，绕该点左右偏转时枪口的位移最大；若绕武器根或中心旋转，会让枪显得“漂浮”。

#### Lerp 插值方案

1. Interp：快速响应、效果不错，效果做好需要更多参数，否则会直来直去，很假
2. Sprint Interp：更有弹性、提供变速运动，让其看起来更生动，但配置复杂，不直观。但确实更适合Weapon Sway
3. Force & Settle & Damp：简化的Sprint Interp，保留了变速运动，参数更直观、不会很弹
4. Lerp+sin: 有人用Sin来实现速度变化：[Whats the best way to make gun sway?](https://devforum.roblox.com/t/whats-the-best-way-to-make-gun-sway/284153/2)

#### Post Process 基本方案

**柏林噪声**\
在限制了最大偏移时能大幅度降低生硬感\
在正常摆动时也能通过轻微的随机数来提升少量观感

**回弹**

> 并没有实现，但效果非常好

无论采用哪种插值方案，都是专注于摇摆的过程。但玩家如果还需要一个晃动完成的确认感表现，当武器归位时增加一个回弹动画会是一个优秀的方案\
SprintInterp也能简单实现回弹效果，但表现上比不上专门的回弹动画，其极高的耦合度也让回弹并不能单独被控制\
如：\
[SKG Shooter Framework](https://youtu.be/hCvq81uiDHU?si=7ej16nyoKlTL1QIV\&t=64) 实现了极为优质的回弹动画\
[LowPolyShooterPack](https://www.fab.com/listings/90ba076a-dc9a-4782-9ac8-dc2ed4f06405)：大部分状态切换完成后都会播放回弹动画

对SwayTarget进行delay / ramp-up

> 未实现

引入一个 **启动 delay + 增强曲线**，特别是在进入瞄准 (ADS) 时。这样视觉效果更自然

### 1.2.3. 外部工程思路

#### 运动状态

考虑不同运动状态下的视觉变化(AI)：

* 行走/跑动时：武器摆动幅度更大，速度更快，可能伴随武器上下小幅“抛物”运动 (bobbing) + sway。
* 站立/瞄准 (ADS)时：幅度缩小，响应延迟加大，甚至增加“惯性”感觉（武器慢慢跟上视角）。
* 蹲伏/匍匐：幅度最小，优雅/稳定感强。
* 跳跃/落地：瞬间武器有冲击位移＋摆动＋转动 →然后恢复。
* 武器切换、换弹、踢脚、滑行等动作也可以用类似机制增强反馈。

> \[!note]- 状态延迟与 Ramp（StateEntryDelay / StateRampTime）\
> 当从跑→瞄或跳→落地，武器参数瞬间变化会感觉突兀。可以做跨状态插值（参数 lerp）或使用 timeline 让变化平滑。(实测意义不大)
>
> ```cpp
> float stateRamp = ComputeStateRamp(Params);
> swayOffsetTarget *= stateRamp;  
> rotTarget = rotTarget * stateRamp;
> float ComputeStateRamp(const FSwayParams& Params) const  
> {  
>     float Elapsed = FMath::Max(0.f, LocalTime - StateStartTime); 
>     if (Elapsed < Params.StateEntryDelay) return 0.f;  
>     float AfterDelay = Elapsed - Params.StateEntryDelay;  
>     if (Params.StateRampTime <= KINDA_SMALL_NUMBER) return 1.f;  
>     return FMath::Clamp(AfterDelay / Params.StateRampTime, 0.f, 1.f);  
> }
> ```

#### 屏幕震动

玩家视角旋转同时可能伴随镜头轻微晃动（camera bob/sway），建议武器摆动与镜头运动 **不同步但联动**。这会增强“手持武器的真实感”。

## 1.3. 插件化

### Obj继承vs DataAsset

DataAsset——不需要每次修改后编译\
可以运行时编辑器实时编辑。修改父类数据后不用重新保存引用的蓝图\
Obj继承——数据+函数 √\
可以单独存储函数中的数据、代码逻辑简单清晰、不需要独立Component/Subsystem用于计算。\
使用和理解没有心智负担

## 1.4. 未处理的资料

[Weapon sway Solved](https://forums.unrealengine.com/t/weapon-sway-solved/326436)

## 1.5. 错误方案

### 1.5.1. Spring Arm

> 几乎不可用，但在大量初学者教程中出现

勾选Enable Camera Rotation Lag，使用全身延迟旋转\
[Weapon Sway in UE4- THE CORRECT WAY | Black Ops Zombies in UE4 #6 - YouTube](https://youtu.be/rZUfkXaVvX4)\
缺点：不可控，无法限制摆动幅度，并不能真正解决问题

# 2. weapon bobbing

### 零散资料

CoD 等 AAA 游戏在 ADS 时给 idle sway 加了 **延迟 (delay)** + 逐渐增强 (ramp up)

### 2.1.1. 基本概念：

武器摆动是枪支的无意移动，由呼吸和手部不稳引起。可通过屏住呼吸来消除，但会消耗体力，或趴在地上。作为平衡机制，通过增加使用高伤害武器（所有机枪和瞄准武器）所需的技能来发挥作用。仅在第一人称瞄准时使用

当玩家静止、瞄准时，武器依然有轻微惯性漂移，如手抖、呼吸带动镜头细微振动。可以用 Perlin Noise 或 Lissajous 曲线生成。\
ADS idle sway 并不是立即开始，而是在用户瞄准后有一个 **延迟 (delay)**，然后慢慢增强到最大强度 (over ~3 秒)

> \[!example] [现代战争中的闲置摇摆控制与瞄准稳定性](https://www.youtube.com/watch?v=ZdnDTJQBNPY)\
> 使命召唤中Idle Sway采用8字型摇晃\
> ![](https://i.ytimg.com/vi/ZdnDTJQBNPY/hqdefault_39966.jpg?sqp=-oaymwEnCNACELwBSFryq4qpAxkIARUAAIhCGAHYAQHiAQoIGBACGAY4AUAB\&rs=AOn4CLDRDY1oo08zXx2W1MhzhhikUJNRKA)

#### 参考内容

[Unity制作的IdleSway效果](https://www.youtube.com/watch?v=1qz7VmQ9s78)

> \[!example]- [reddit 关于Idle武器晃动的讨论](https://www.reddit.com/r/gamedev/comments/3m548h/techniques_for_weapon_sway_in_fps)
>
> > \[!quote]+\
> > 到目前为止，我看到 Lissajous 曲线可能是武器晃动的不错选择，结合基于鼠标移动强度的阻尼效果。但我也看到有些人使用 Perlin 噪声来处理这个问题...为什么会这样？如何用 Perlin 获得良好的曲线？\
> > 我还注意到一些老式射击游戏，如 Quake，只是通过插值生成一个 8 字形状。
>
> > \[!quote]+\
> > 在我的游戏中实现武器晃动（我没有动画）时，我使用了两个曲线。它们基本上看起来像一条横向的 S 形（我不擅长数学）。其中一个曲线的持续时间比另一个稍短，并且是倒置的，所以晃动会有一些变化。较短的曲线与时间轴关联，基本上控制垂直变换，而较长的曲线则控制水平方向。考虑到我的射击逻辑直接来自枪管的正前方，这增加了很多变化性，而幸运的是也提供了一些可预测性。我还需要提到的是，这是在 Unreal Blueprints 中实现的，所以我不知道在其他引擎中是否存在类似的功能。

> \[!example]+ [Techniques for weapon sway in FPS](https://www.reddit.com/r/gamedev/comments/3m548h/techniques_for_weapon_sway_in_fps/?utm_source=chatgpt.com)

[Dayz的idle sway](https://www.youtube.com/watch?v=gWX-LriRL-c)\
[战地6冬季攻势和普通动画区别](https://youtube.com/shorts/5jKKFe5g0tI?si=FmZtNGytnfeIAKDH)

# 3. Shoot Sway

[How I Made My FPS Game Feel Better To Play](https://www.youtube.com/watch?v=SWf9pNdivpo\&t=8s)\
视频中一开始采用随机晃动让人眩晕，后来采用purlin noise获得平滑的摄像机抖动![](https://gcore.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-28_11-02-18.gif)

[自制COD20 优化开火动画](https://www.bilibili.com/video/BV1dRffYjEra)

# 4. 移动摇晃

[Realistic Headbob Camera Shakes](https://www.youtube.com/watch?v=lwM-SbDKnxQ)
