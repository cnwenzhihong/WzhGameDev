---
created: 2025-11-22T21:19:31+08:00
modified: 2025-12-24T20:38:46+08:00
feature: thumbnails/external/e6edcd6b8422c78b3b4b64b340d7848d.png
thumbnail: thumbnails/resized/37f015824f567075ec9bf12a78536396_b89e22fb.jpg
tags:
  - 3C
  - FPS
  - blog
dg-publish: "true"
dg-home: "true"
cssclasses:
  - list-cards
title: FPS中的武器摇晃
date: 2025-11-22T13:19:29.880Z
lastmod: 2025-12-24T12:38:47.548Z
---
# 旋转摇晃Weapon Sway

## 代码实现

我们可以将其分为：获取转动输入→计算偏移量→表现三个步骤

> 先获取用户转动差值，做各种处理后，用于表现效果

### 一、处理输入量

> 计算转动量/输入量的速度，用于后续处理

#### 1.1 原始数据采样——转动量

类似于TPS的瞄准偏移，但TPS关注角度，而FPS更在意变化量

##### 鼠标输入 ★★★

理论上是最直接简单的获取方式，完全不需要计算\
不容易获得精确数值(要么0要么1)。\
UE愈加复杂的输入系统让实现不再简单。

> \[!example]- 实现\
> 0.1秒间隔，无太大必要优化，并未处于deltaSeconds\
> ![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251126063200443.png)

##### 旋转差值 ★★★★★

计算`ControllerRotation`的旋转差值\
可以抹平不同鼠标Dpi的差距，获取差值也简单，需要单独保存上一帧的旋转量\
![|300](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251216011431739.png)

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

如果想平滑输入，可以对角速度进行低通滤波，会看起来更舒缓，但变化较慢。\
适用场景：

* 大幅度摆动时，防止短时间参数跳变引起Target跳变
* Interp平滑曲线，几乎必备

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
> ![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251215013753758.png)\
> 视觉效果：\
> Normal:\ <video src="https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/NoLowpass.mp4" controls></video>\
> Lowpass\ <video src="https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/Lowpass.mp4" controls></video>

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

小幅度转动时提供更小的视觉变化，大幅度转动时再提升效果往往能带来更好的效果。指数函数能较低成本地实现。\
![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251221161521741.png)

> \[!example]- 示例
>
> ```cpp
> //可以改为曲线
> auto NonLinear = [](float v) { return FMath::Sign(v) * FMath::Pow(FMath::Abs(v), 1.15f); };  
> float nx = NonLinear(rotSpeedNorm.X);  
> float ny = NonLinear(rotSpeedNorm.Y);
> ```

### 二、计算偏移量

我们需要得到WeaponSway的关键结果：`SwayOffset`和`SwayRot`

#### 2.1 计算Sway Target

通过上面获取的输入量(偏移/速度/加速度)，处理得到想要的偏移和旋转目标值。

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

##### 左右旋转增加值

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

#### 2.2 对Sway Target插值

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

### 三、应用参数

> \[!example]- 上文完整代码：
>
> ```cpp
> void UPawnComp_FPSRotWeaponSway::TickWeaponSway(float DeltaSeconds, const FRotator& ControlRotation, ESwayState NewState)  
> {  
>     if (DeltaSeconds <= 0.f) return;  
>     LocalTime += DeltaSeconds;  
>     // State transitions  
>     if (NewState != CurrentState)  
>     {       CurrentState = NewState;  
>        StateStartTime = LocalTime;  
>     }  
>   
>     FSwayParams const& Params = GetParamsForState(CurrentState);  
>   
>   
>     // 1) delta rot  
>     FRotator deltaRot = UKismetMathLibrary::NormalizedDeltaRotator(ControlRotation, LastControlRot);  
>     LastControlRot = ControlRotation;  
>   
>   
>     // 2) rot speed (deg/s) and smoothing  
>     FVector2D rotSpeedFrame(deltaRot.Yaw / DeltaSeconds, deltaRot.Pitch / DeltaSeconds);  
>     // low-pass smoothing  
>     if (bUseInputLowpass)  
>     {       float alpha = FMath::Clamp(DeltaSeconds * 10.f, 0.f, 1.f);  
>        RotSpeedFiltered.X = FMath::Lerp(RotSpeedFiltered.X, rotSpeedFrame.X, alpha);  
>        RotSpeedFiltered.Y = FMath::Lerp(RotSpeedFiltered.Y, rotSpeedFrame.Y, alpha);  
>     }    else  
>     {  
>        RotSpeedFiltered.X=rotSpeedFrame.X;  
>        RotSpeedFiltered.Y=rotSpeedFrame.Y;  
>     }    // accel  
>     FVector2D rotAccel((RotSpeedFiltered.X - PrevRotSpeedFiltered.X) / DeltaSeconds,  
>                        (RotSpeedFiltered.Y - PrevRotSpeedFiltered.Y) / DeltaSeconds);  
>     PrevRotSpeedFiltered.X = RotSpeedFiltered.X;  
>     PrevRotSpeedFiltered.Y = RotSpeedFiltered.Y;  
>   
>   
>     // 3) normalize  
>     FVector2D rotSpeedNorm(RotSpeedFiltered.X / Params.FiducialRotSpeed, RotSpeedFiltered.Y / Params.FiducialRotSpeed);  
>     rotSpeedNorm.X = FMath::Clamp(rotSpeedNorm.X, -1.f, 1.f);  
>     rotSpeedNorm.Y = FMath::Clamp(rotSpeedNorm.Y, -1.f, 1.f);  
>   
>   
>     // 4) nonlinear mapping (emphasize higher speeds)  
>     //改为曲线，可以手动选择曲线  
>     auto NonLinear = [](float v) { return FMath::Sign(v) * FMath::Pow(FMath::Abs(v), 1.15f); };  
>     float nx = NonLinear(rotSpeedNorm.X);  
>     float ny = NonLinear(rotSpeedNorm.Y);  
>           
>       
> // 5) compute translation target  
>     FVector swayOffsetTarget = FVector(  
>        nx * Params.SwayOffsetMulti.X * -1.f,  
>        nx * Params.SwayOffsetMulti.Y,  
>        ny * Params.SwayOffsetMulti.Z  
>     );  
>   
>     // 6) compute rotation target (separate)  
>     FRotator swayRotTarget = FRotator(  
>        nx * Params.SwayRotMulti.Pitch,  
>        nx * Params.SwayRotMulti.Yaw,  
>        ny * Params.SwayRotMulti.Roll * -1.f  
>     );  
>
>     // 9) position spring interp    SwayOffset = UKismetMathLibrary::VectorSpringInterp(SwayOffset, swayOffsetTarget, PosSpringState,  
>                                                         Params.Pos_Stiffness, Params.Pos_Damping, DeltaSeconds,  
>                                                         Params.Pos_Mass,Params.TargetVelocityAmount);  
>     // clamp  
>     if (SwayOffset.Size() > Params.MaxOffset)  
>     {       SwayOffset = SwayOffset.GetSafeNormal() * Params.MaxOffset;  
>     }  
>     // 10) rotation spring interp  
>     SwayRot = RotatorSpringInterp(SwayRot, swayRotTarget, RotSpringState, Params.Rot_Stiffness, Params.Rot_Damping,  
>                                   DeltaSeconds, Params.Rot_Mass,Params.MaxRot);  
> }
>
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

#### 基本使用

在Tick中使用\
![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251218041755810.png)

将计算出的参数用于修改武器IK骨骼即可\
![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251218035612026.png)

#### 拓展

##### 改变旋转中心

有的文章中提到，Yaw轴旋转可以设置为绕枪托旋转效果会更好\
下图以枪口为轴枢点更方便理解：\
![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/weaponsway_yawpivot.png)

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
![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251218042239133.png)

> \[!example]- 效果\
> 枪托为中心(Pivot=(0,-5,0)) <video src="https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/2025-12-18%2005-16-05.mp4" controls></video>\
> 枪头为中心(Pivot=(0,10,0))<video src="https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/2025-12-18%2005-19-39.mp4" controls></video>

## 参数配置

### 优秀的实现案例

[【自制COD20 ?】开发日志5-绕任意轴旋转枪身](https://www.bilibili.com/video/BV14UfZYaEQn)：左右旋转采用三轴共转能实现极为大开大合的效果

#### 3.1 变化速率

##### 平滑速率

一般游戏采用方案

##### 先慢后快

需要又稳重又不让感觉拖沓的环境下(CS)，鼠标停下后立刻加快插值速度，营造确认感。

> \[!note]- 增强方案
>
> * CriticalDamping 通常设为 `2 * sqrt(Stiffness * Mass)` 才为临界阻尼，设置不同值，可得到欠阻尼（震荡）或过阻尼（缓慢）效果。建议理解该物理背景。
> * 可以尝试的参数配置：Stiffness 中等偏高（快速响应但不过火）、CriticalDamping 多数接近 1（轻微震荡但不发狂）、Mass 视武器重量或状态而定（重武器 Mass大→摆动慢）。
> * 大 Mass +大 Stiffness 可能导致高频震荡或振幅异常。
> * 每帧确认 deltaSeconds 非常小的情况下数值稳定性，避免弹簧公式爆炸（如果 stiffness or mass 过大且 delta small）。可做最大 delta clamp。

> \[!note] 增强方案\
> 最好的效果：玩家向右转，武器向左并稍微上/下摆动、同时绕自身轻微旋转
>
> * swayOffsetMulti用 **速度曲线（非线性）** 替代：比如用速度^n 或平滑函数（例如 SmoothStep）以便高转速带更强但低转速微弱。
> * 为偏移和旋转分别设计 **不同方向响应**：例如 yaw→横向 + roll，pitch→纵向 + pitch 轻微。

> \[!note]-  拓展建议\
> **fiducialRotSpeed 可以根据不同模式（走、跑、蹲、瞄准）动态改变**。跑动时武器晃动更强，瞄准时更弱。

### 常见游戏方案

从大小来看有超前和追赶两种状态，前者看起来更灵敏，后者看起来更真实\
从移动方式来看，围绕角色旋转、围绕右手旋转、围绕右手偏移

#### 围绕角色旋转：

出现于很多老游戏中，如虚幻竞技场，一般用Lag就能实现\
容易实现，视野变化大，一般搭配角色追随摄像机效果，体现沉重感

##### 示例

> \[!example]- cs2\
> 采用Lag，少量延迟，停下时立刻追赶上，给人感觉并不拖沓\
> ![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/RotSway_CS2.mp4)

> \[!example]- UE示例\
> https://forums.unrealengine.com/t/weapon-sway-solved\
> [TFE Game - Rotation Lag/Leading](https://www.youtube.com/watch?v=6OFmhQ55iJo)\
> 我发现除了旋转滞后/领先之外，如果你同时根据滞后/领先值做一些相对位置偏移，效果会好很多。也就是说，我当前的 UpdateWeaponLoc 函数如下：
>
> ```cpp
>   void APWNWeapon::UpdateLocRot(FVector newLoc, FRotator newRot, float deltaTime) { FRotator finalRot = newRot; 
>   // Rotation Leading/Lag
>   float lagDiff; finalRot.Pitch = LagWeaponRotation(newRot.Pitch, LastRotation.Pitch, deltaTime, MaxPitchLag, 1, lagDiff); 
>   finalRot.Yaw = LagWeaponRotation(newRot.Yaw, LastRotation.Yaw, deltaTime, MaxYawLag, 0, lagDiff); finalRot.Roll = newRot.Roll; 
>   // Location offset 
>   float locDiff = lagDiff / MaxYawLag; locDiff *= MaxLocLagX; 
>   ArmMesh->SetRelativeLocation(FVector(ArmMeshOffset.X, ArmMeshOffset.Y + locDiff, ArmMeshOffset.Z), false); 
>   // Set our loc 
>   LastRotation = newRot; this->SetActorLocation(newLoc); this->SetActorRotation(finalRot); 
> ```

![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/TFE%20Game%20-%20Rotation%20Lag%E2%A7%B8Leading.mp4)

#### 围绕手/武器旋转

大量游戏和教程采用\
容易实现，视角变化适中，大都为超前效果，看起来相应很快，应用于双手步枪有非常好的效果。

##### 原则

一般左右旋转控制Yaw轴、上下旋转控制Pitch轴、左右移动控制Roll轴，但左右旋转控制Roll在减少视野变化的前提下效果也不错\
左右超前，上下追赶也许更贴近人的潜意识\
真实武器摆动里 yaw 幅度比 pitch/roll 小，roll（武器绕镜）更能体现质感。

##### 示例

超前\
\[tabs]

* UE5 教程1\
  https://www.youtube.com/watch?v=rZUfkXaVvX4\
  教程中提到，给模型添加的SpringArm的方案效果并不好，因为无法限制他、并且很晃\
  ![|350](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-26_07-20-24.gif)
* UE5 教程2\
  https://www.youtube.com/watch?v=YahaZR6Umxo\
  ![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-26_07-23-44.gif)
* UE5 教程3\
  https://www.youtube.com/watch?v=eYsjiw0fXvU\
  视频中提到，我们可以反转参数来实现追赶效果。\
  ![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-27_01-33-44.gif)
* UE 实例1\
  https://www.youtube.com/watch?v=cpPOA6yKsoo\
  左右移动时采用roll旋转，鼠标X旋转用较小的Yaw，整体效果非常优秀\
  ![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-27_17-35-54.mp4)

追赶：\
\[tabs]

* UE Example\
  [Realistic Weapon Sway In UE5](https://www.youtube.com/watch?v=MMvBRQtjFmQ)\
  ![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-27_01-30-43.gif)
*

#### 围绕手/武器偏移

https://www.youtube.com/watch?v=u9SuIsd7Dlw

##### 示例

* 军团要塞2\
  [Weapon Sway in Team Fortress 2 Again](https://www.youtube.com/watch?v=4wPAzutIVpc)\
  ![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-27_16-40-17.gif)\
  [UFPS - Procedural camera & animation system intro (2012)](https://www.youtube.com/watch?v=Hu0v0kTWH7w)

#### 围绕手/武器旋转+偏移

可以得到最符合直觉(不一定表现最强)的质感：玩家向右猛转→武器轻微绕镜顺转+向左偏移

### 工程化

#### 武器是“追随”视角还是“超前”视角？

有些游戏武器微微滞后（Lag）给人惯性感觉，有些武器反而“提前”给玩家视觉响应（Lead）以增强 responsiveness。根据游戏风格定。论坛中曾建议“leading”可以感觉更爽快。

#### 运动状态

考虑不同运动状态下的视觉变化：\
\[timeline]

* 行走/跑动时：武器摆动幅度更大，速度更快，可能伴随武器上下小幅“抛物”运动 (bobbing) + sway。
* 站立/瞄准 (ADS)时：幅度缩小，响应延迟加大，甚至增加“惯性”感觉（武器慢慢跟上视角）。
* 蹲伏/匍匐：幅度最小，优雅/稳定感强。
* 跳跃/落地：瞬间武器有冲击位移＋摆动＋转动 →然后恢复。
* 武器切换、换弹、踢脚、滑行等动作也可以用类似机制增强反馈。\
  当从跑→瞄或跳→落地，武器参数瞬间变化会感觉突兀。可以做跨状态插值（参数 lerp）或使用 timeline 让变化平滑。

#### 屏幕震动

玩家视角旋转同时可能伴随镜头轻微晃动（camera bob/sway），建议武器摆动与镜头运动 **不同步但联动**。这会增强“手持武器真实”感。

### 旋转IK\_Gun的Yaw/Pitch

最简单、最容易理解\
能达到的最好效果：\
[SKG Shooter Framework](https://www.youtube.com/watch?v=hCvq81uiDHU\&list=PLnHeglBaPYu_BueK4IiXg34u5pGpsDkJj):采用SpringLerp，添加了一个类似LowPolyShooter的回弹效果

### 额外效果

[# SKG Shooter Framework](https://www.youtube.com/watch?v=hCvq81uiDHU\&list=PLnHeglBaPYu_BueK4IiXg34u5pGpsDkJj):添加了一个类似LowPolyShooter的回弹效果\
[COD2的武器摇摆](https://www.youtube.com/watch?v=kjVSLsz8dCs\&source_ve_path=MTc4NDI0)\
获取输入量，使用动画Yaw和Pitch旋转

<iframe width="100%" height="200px"  src="https://blueprintue.com/render/mw-zlw-z/" scrolling="no" allowfullscreen></iframe>

## 终极方案

DataAsset比Obj继承好的地方：可以编辑器实时编辑。修改父类数据后不用重新保存引用的蓝图\
Obj好的地方：可以单独存储函数中的数据、代码逻辑简单清晰、不需要独立Component用于计算

### 代码

```cpp
void UPawnComp_FPSRotWeaponSway::TickWeaponSway(float DeltaSeconds, const FRotator& ControlRotation, bool bHasInput,  
                                                ESwayState NewState)  
{  
    if (DeltaSeconds <= 0.f) return;  
    LocalTime += DeltaSeconds;  
  
  
    // State transitions  
    if (NewState != CurrentState)  
    {       CurrentState = NewState;  
       StateStartTime = LocalTime;  
    }  
  
    FSwayParams const& Params = GetParamsForState(CurrentState);  
  
  
    // 1) delta rot  
    FRotator deltaRot = UKismetMathLibrary::NormalizedDeltaRotator(ControlRotation, LastControlRot);  
    LastControlRot = ControlRotation;  
  
  
    // 2) rot speed (deg/s) and smoothing  
    FVector2D rotSpeedFrame(deltaRot.Yaw / DeltaSeconds, deltaRot.Pitch / DeltaSeconds);  
    // low-pass smoothing  
    float alpha = FMath::Clamp(DeltaSeconds * 10.f, 0.f, 1.f);  
    RotSpeedFiltered.X = FMath::Lerp(RotSpeedFiltered.X, rotSpeedFrame.X, alpha);  
    RotSpeedFiltered.Y = FMath::Lerp(RotSpeedFiltered.Y, rotSpeedFrame.Y, alpha);  
    // accel  
    FVector2D rotAccel((RotSpeedFiltered.X - PrevRotSpeedFiltered.X) / DeltaSeconds,  
                       (RotSpeedFiltered.Y - PrevRotSpeedFiltered.Y) / DeltaSeconds);  
    PrevRotSpeedFiltered.X = RotSpeedFiltered.X;  
    PrevRotSpeedFiltered.Y = RotSpeedFiltered.Y;  
  
  
    // 3) normalize  
    FVector2D rotSpeedNorm(RotSpeedFiltered.X / Params.FiducialRotSpeed, RotSpeedFiltered.Y / Params.FiducialRotSpeed);  
    rotSpeedNorm.X = FMath::Clamp(rotSpeedNorm.X, -1.f, 1.f);  
    rotSpeedNorm.Y = FMath::Clamp(rotSpeedNorm.Y, -1.f, 1.f);  
  
  
    // 4) nonlinear mapping (emphasize higher speeds)  
    //改为曲线，可以手动选择曲线  
    auto NonLinear = [](float v) { return FMath::Sign(v) * FMath::Pow(FMath::Abs(v), 1.15f); };  
    float nx = NonLinear(rotSpeedNorm.X);  
    float ny = NonLinear(rotSpeedNorm.Y);  
    // 5) compute translation target  
    FVector swayOffsetTarget = FVector(  
       nx * Params.SwayOffsetMulti.X * -1.f,  
       nx * Params.SwayOffsetMulti.Y,  
       ny * Params.SwayOffsetMulti.Z  
    );  
  
  
    // 6) compute rotation target (separate)  
    FRotator rotTarget = FRotator(  
       ny * Params.SwayRotMulti.Pitch * -1.f,  
       nx * Params.SwayRotMulti.Yaw * -1.f,  
       nx * Params.SwayRotMulti.Roll  
    );  
  
  
    // 7) lead/lag handling  
    float lagFactor = FMath::Clamp(Params.LeadLag, -1.f, 1.f);  
    if (lagFactor > 0.f)  
    {       // lag via 1st-order low pass  
       float tau = FMath::Lerp(0.02f, 0.5f, lagFactor);  
       float k = DeltaSeconds / (tau + DeltaSeconds);  
       TargetOffsetFiltered = FMath::Lerp(TargetOffsetFiltered, swayOffsetTarget, k);  
       TargetRotFiltered = FMath::Lerp(TargetRotFiltered, FVector(rotTarget.Pitch, rotTarget.Yaw, rotTarget.Roll), k);  
       swayOffsetTarget = TargetOffsetFiltered;  
       rotTarget = FRotator(TargetRotFiltered.X, TargetRotFiltered.Y, TargetRotFiltered.Z);  
    }    //问题，数值不影响最终效果  
    else if (lagFactor < 0.f)  
    {       // small predictive lead using angular accel  
       float leadAmp = FMath::Abs(lagFactor);  
       swayOffsetTarget += FVector(rotAccel.X, 0.f, rotAccel.Y) * Params.SwayOffsetMulti * leadAmp * 0.02f;  
       rotTarget.Pitch += rotAccel.Y * Params.SwayRotMulti.Pitch * leadAmp * 0.01f;  
    }  
  
    // 8) state ramp and idle noise  
    float stateRamp = ComputeStateRamp(Params);  
    // if (!bHasInput && FVector2D(rotSpeedNorm).SizeSquared() < 0.01f)  
    // {    //     FVector noise = GetIdleNoise(LocalTime) * Params.IdleNoiseAmp;    //     swayOffsetTarget += noise * 0.3f * stateRamp;    //     rotTarget += FRotator(noise.Z * Params.IdleNoiseRotAmp * 0.2f, noise.X * Params.IdleNoiseRotAmp * 0.15f,    //                           noise.Y * Params.IdleNoiseRotAmp * 0.25f);    // }  
  
    swayOffsetTarget *= stateRamp;  
    rotTarget = rotTarget * stateRamp;  
  
  
    // 9) position spring interp  
    SwayOffset = UKismetMathLibrary::VectorSpringInterp(SwayOffset, swayOffsetTarget, PosSpringState,  
                                                        Params.Pos_Stiffness, Params.Pos_Damping, DeltaSeconds,  
                                                        Params.Pos_Mass);  
  
  
    // 10) rotation spring interp  
    SwayRot = RotatorSpringInterp(SwayRot, rotTarget, RotSpringState, Params.Rot_Stiffness, Params.Rot_Damping,  
                                  DeltaSeconds, Params.Rot_Mass);  
  
    // 11) clamp  
    if (SwayOffset.Size() > Params.MaxOffset) SwayOffset = SwayOffset.GetSafeNormal() * Params.MaxOffset;  
    SwayRot.Pitch = FMath::Clamp(SwayRot.Pitch, -Params.MaxRot.Pitch, Params.MaxRot.Pitch);  
    SwayRot.Yaw = FMath::Clamp(SwayRot.Yaw, -Params.MaxRot.Yaw, Params.MaxRot.Yaw);  
    SwayRot.Roll = FMath::Clamp(SwayRot.Roll, -Params.MaxRot.Roll, Params.MaxRot.Roll);  
}
```

### 步骤

#### 1. 计算瞬时角速度

得到当前帧相对上一帧的欧拉角差（通常取 Yaw 和 Pitch），用于计算瞬时角速度（deg/s）。

```cpp
FRotator deltaRot = UKismetMathLibrary::NormalizedDeltaRotator(ControlRotation, LastControlRot);  
LastControlRot = ControlRotation;
```

NormalizedDeltaRotator: 将差异角度控制在\[-180, 180]之间

#### 2. 低通滤波 / 平滑角速度/加速度（RotSpeedFiltered）

对每帧测得的角速度做指数平滑（低通），去掉抖动 / 鼠标抖动(0.1s内)造成的高频噪声。

```cpp
FVector2D rotSpeedFrame(deltaRot.Yaw / DeltaSeconds, deltaRot.Pitch / DeltaSeconds);  
  
// low-pass smoothing  
float alpha = FMath::Clamp(DeltaSeconds, 0.f, .1f);
RotSpeedFiltered.X = FMath::Lerp(RotSpeedFiltered.X, rotSpeedFrame.X, alpha);  
RotSpeedFiltered.Y = FMath::Lerp(RotSpeedFiltered.Y, rotSpeedFrame.Y, alpha);  
  
// accel  
FVector2D rotAccel((RotSpeedFiltered.X - PrevRotSpeedFiltered.X) / DeltaSeconds,  
                   (RotSpeedFiltered.Y - PrevRotSpeedFiltered.Y) / DeltaSeconds);  
PrevRotSpeedFiltered = RotSpeedFiltered;  
```

#### 3. 归一化角速度（fiducialRotSpeed）

把角速度映射到 \[-1,1] 范围,玩家鼠标灵敏度差异很大，归一化允许统一调参（例如把 180°/s 视为“满值”）

```cpp
FVector2D rotSpeedNorm(RotSpeedFiltered.X / Params.FiducialRotSpeed, RotSpeedFiltered.Y / Params.FiducialRotSpeed);  
rotSpeedNorm.X = FMath::Clamp(rotSpeedNorm.X, -1.f, 1.f);  
rotSpeedNorm.Y = FMath::Clamp(rotSpeedNorm.Y, -1.f, 1.f);
```

FiducialRotSpeed: 每秒转速，可以设为1440，4圈

#### 4. 非线性映射（例如 pow/sign）

把归一化速度做非线性压缩或放大（低速更线性/柔和、高速更敏感），用来塑造手感曲线。微小头部移动只想要极小响应，而大幅快速转头希望响应更加明显。非线性函数（如 pow）能实现这种“阈值感”。

```cpp
auto NonLinear = [](float v) { return FMath::Sign(v) * FMath::Pow(FMath::Abs(v), 1.15f); };  
float nx = NonLinear(rotSpeedNorm.X);  
float ny = NonLinear(rotSpeedNorm.Y);
```

NonLinear:可以换成曲线映射\[0,1]\
Pow: 可以调更高或设置为参数

#### 5. 计算目标偏移与旋转（swayOffsetTarget / rotTarget）

在真实枪械中，Pitch、Roll、Yaw 的来源与幅度不同（例如 yaw 小，roll 明显），把它们分开能分别调节惯性与幅度。

```cpp
// 5) compute translation target  
FVector swayOffsetTarget = FVector(  
    nx * Params.SwayOffsetMulti.X * -1.f,  
    nx * Params.SwayOffsetMulti.Y,  
    ny * Params.SwayOffsetMulti.Z  
);  
// 6) compute rotation target (separate)  
FRotator rotTarget = FRotator(  
    ny * Params.SwayRotMulti.Pitch * -1.f,  
    nx * Params.SwayRotMulti.Yaw * -1.f,  
    nx * Params.SwayRotMulti.Roll  
);
```

#### 6. Lead / Lag（滞后或超前）处理

根据 `Params.LeadLag`，做一阶低通（lag）或简单预测（lead）。\
控制武器对视角的响应是“稍滞后”（更真实/有惯性）还是“略超前”（更敏捷/竞技感）。

* Lag：武器在转动停止后会有尾巴（overshoot + decay） → 更真实但反应稍慢；
* Lead：能让瞄准看起来更准/快速（适合竞技风格）；

```cpp
float lagFactor = FMath::Clamp(Params.LeadLag, -1.f, 1.f);  
if (lagFactor > 0.f)  
{  
    // lag via 1st-order low pass  
    float tau = FMath::Lerp(0.02f, 0.5f, lagFactor);  
    float k = DeltaSeconds / (tau + DeltaSeconds);  
    TargetOffsetFiltered = FMath::Lerp(TargetOffsetFiltered, swayOffsetTarget, k);  
    TargetRotFiltered = FMath::Lerp(TargetRotFiltered, FVector(rotTarget.Pitch, rotTarget.Yaw, rotTarget.Roll), k);  
    swayOffsetTarget = TargetOffsetFiltered;  
    rotTarget = FRotator(TargetRotFiltered.X, TargetRotFiltered.Y, TargetRotFiltered.Z);  
}  
else if (lagFactor < 0.f)  
{  
    // small predictive lead using angular accel  
    float leadAmp = FMath::Abs(lagFactor);  
    swayOffsetTarget += FVector(rotAccel.X, 0.f, rotAccel.Y) * Params.SwayOffsetMulti * leadAmp * 0.02f;  
    rotTarget.Pitch += rotAccel.Y * Params.SwayRotMulti.Pitch * leadAmp * 0.01f;  
}
```

#### 8. 状态延迟与 Ramp（StateEntryDelay / StateRampTime）

在状态切换（如进入 ADS）时给出延迟和渐入（例如瞄准后不是瞬间从“跑步晃动”变为“稳态”，而是有个缓慢过渡）。

```cpp
float stateRamp = ComputeStateRamp(Params);
swayOffsetTarget *= stateRamp;  
rotTarget = rotTarget * stateRamp;
float ComputeStateRamp(const FSwayParams& Params) const  
{  
    float Elapsed = FMath::Max(0.f, LocalTime - StateStartTime); 
    if (Elapsed < Params.StateEntryDelay) return 0.f;  
    float AfterDelay = Elapsed - Params.StateEntryDelay;  
    if (Params.StateRampTime <= KINDA_SMALL_NUMBER) return 1.f;  
    return FMath::Clamp(AfterDelay / Params.StateRampTime, 0.f, 1.f);  
}
```

#### 9. 弹簧插值（VectorSpringInterp）

用弹簧阻尼模型使当前位置朝目标平滑过渡，并能表现出惯性、超调（如果阻尼不足）或快速回弹（如果阻尼大）。\
弹簧模型比简单的 LERP 更接近物理，能自然模拟质量、刚度、阻尼的不同组合，产生丰富的“尾巴/回弹”效果。

```cpp
SwayOffset = UKismetMathLibrary::VectorSpringInterp(SwayOffset,swayOffsetTarget, PosSpringState, Params.Pos_Stiffness, Params.Pos_Damping, DeltaSeconds,  Params.Pos_Mass);
SwayRot = RotatorSpringInterp(SwayRot, rotTarget, RotSpringState, Params.Rot_Stiffness, Params.Rot_Damping, DeltaSeconds, Params.Rot_Mass);

```

旋转弹簧插值：社区推荐用自己的弹簧求解[Spring-based Animation and Quaternions](https://toqoz.fyi/springs.html?utm_source=chatgpt.com)

#### 10. 限幅（MaxOffset / MaxPitch/Yaw/Roll）

保护视觉（防止武器飘出画面或产生穿模），并保证在极端输入下效果可控。\
用户灵敏度、鼠标抖动或跳帧可能造成瞬时超大值，限幅能保护不破坏玩家视觉体验。上述情况都很容易出现

```cpp
if (SwayOffset.Size() > Params.MaxOffset) SwayOffset = SwayOffset.GetSafeNormal() * Params.MaxOffset;  
SwayRot.Pitch = FMath::Clamp(SwayRot.Pitch, -Params.MaxRot.Pitch, Params.MaxRot.Pitch);  
SwayRot.Yaw = FMath::Clamp(SwayRot.Yaw, -Params.MaxRot.Yaw, Params.MaxRot.Yaw);  
SwayRot.Roll = FMath::Clamp(SwayRot.Roll, -Params.MaxRot.Roll, Params.MaxRot.Roll);
```

#### ButtStockPivot（Yaw 绕枪托旋转）在 AnimBP 中的实现

AnimBP 中 `Transform (Modify) Bone` 节点对 `ButtStockPivot` 应用 `SwayRot.Yaw`，而 Pitch/Roll 应用在 WeaponBody/root。\
把绕 Yaw 的旋转枢轴定位在枪托（贴肩处），保证枪托相对稳定，枪口移动最大，符合现实杠杆效果。\
**为什么**：现实中枪托靠肩为固定支点，绕该点左右偏转时枪口的位移最大；若绕武器根或中心旋转，会让枪显得“漂浮”。

## 未处理的资料

[Weapon sway Solved](https://forums.unrealengine.com/t/weapon-sway-solved/326436)

### GPT建议

| **你的实现**                                                            | **AAA /研究 /行业参考**                                                                                                                                                                                                                          | **建议 & 调整方向**                                                                                         |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| 使用鼠标转向速度 (deltaRot) 计算转速 → 归一化 → 目标偏移 (swayOffset\_Target) → 弹簧插值恢复 | 真实游戏 /研究中常用弹簧插值 + 惯性 + 超调 + lag/lead                                                                                                                                                                                                       | 保持弹簧插值，但调参数 (Stiffness, Mass, Damping) 来实现 **超调 + 回弹**。考虑你希望武器在停止输入后有“惯性尾巴 (trail)” + 轻微 overshoot。   |
| 偏移 (swayOffset) 驱动旋转 (swayRot)（简单线性系数映射）                            | AAA 项目中旋转 (pitch / yaw / roll) 往往与速度 +惯性分开处理，有自己的惯性插值                                                                                                                                                                                      | 把旋转 (swayRot) 用自己的弹簧插值 (或类似) 系统，而不是直接从偏移映射。这样旋转也有惯性感觉 (滞后/lead + 回弹)。特别是 roll（武器绕镜转）可以很大程度提升真实感。      |
| 早期退出逻辑：如果无输入且偏移、旋转近零就不计算                                            | AAA 设计中，他们通常不会完全关掉 sway 恢复 / idle 状态 — 武器在静止时也可能有 /启动 idle sway                                                                                                                                                                            | 改进早退策略：即使无输入，也持续更新直到偏移 & 旋转真正回中性。加入 **idle sway** (微小自然晃动)，而不是硬性归零。                                   |
| 参数固定 (swayOffsetMulti、旋转倍率、多项弹簧参数)                                  | AAA 游戏通常根据 “武器类型 +玩家姿态 (站/蹲/跑) +附件” 调整 sway 参数 (Vigor 是一个例子)                                                                                                                                                                               | 将你的系统做成 **状态驱动 (state-based)**：不同姿态 (跑 /站 /瞄准 /蹲) 使用不同参数集 (偏移倍率、弹簧刚度、最大转速等)。这样你可以模仿 AAA 级别多样性的武器手感。   |
| 只用角速度 (转速) 来决定目标偏移                                                  | 研究 /现实中武器响应玩家视角转动时可能有 “leading / lagging” + 超调 + 惯性恢复                                                                                                                                                                                      | 引入 “lead vs lag” 参数：你可以计算目标偏移不仅基于速度，还考虑 “前一帧偏移 (或旋转)” 的惯性趋势 (例如让武器略滞后然后缓慢回弹)，模拟惯性 + 质量感。              |
| 不考虑 “delay / ramp-up”                                               | CoD 等 AAA 游戏在 ADS 时给 idle sway 加了 **延迟 (delay)** + 逐渐增强 (ramp up)                                                                                                                                                                          | 给你的 swayTarget 或旋转目标引入一个 **启动 delay + 增强曲线**，特别是在进入瞄准 (ADS) 时。这样视觉效果更自然，也更贴近 AAA 设计。                  |
| 没有限制最大偏移 /旋转 (或你自己 clamp)                                           | AAA 游戏为了避免视觉偏移太过夸张，一般会对摆动幅度 (translation + rotation) 设上限                                                                                                                                                                                   | 明确定义最大偏移和最大旋转 (sway 限值)，防止在高速转向或极端情况时武器 “飞出画面” 或摆动过度。还可以让这些最大值随武器类别 /状态动态变化。                          |
| 不考虑武器重量 /惯性随武器差异                                                    | 社区 (如 Escape from Tarkov 玩家)经常提到 “武器重量 (mass) 应影响 sway /惯性”。 ([Reddit](https://www.reddit.com/r/EscapefromTarkov/comments/fgohpd?utm_source=chatgpt.com "Weight should directly influence weapon sway"))                                   | 如果你的游戏里有不同武器 (轻步枪、重机枪等)，可以将 **Mass** (弹簧质量) 参数与武器重量 (或重量感) 关联，使 “重武器摇晃慢、惯性大” 这样更真实。                   |
| 没有加入自然微抖 (noise)                                                    | 社区 /开发者经常使用 Perlin 噪声 / Lissajous 曲线来模拟 idle sway。 r/gamedev 上就有提到。 ([Reddit](https://www.reddit.com/r/gamedev/comments/3m548h/techniques_for_weapon_sway_in_fps/?utm_source=chatgpt.com "Techniques for weapon sway in FPS : r/gamedev")) | 加入 **程序噪声 (Perlin noise) 或 Lissajous 曲线** 来生成 idle 摆动 (手抖 /呼吸效果)，让静止时武器也不死板。结合你弹簧系统做一个混合机制 (惯性 + 噪声)。 |

#### 三、小结 + 推荐下一步

* \[ ] **加入更丰富的动态 (延迟、超调、lead/lag、惯性恢复)**，大幅提升逼真感 /重量感。
* \[ ] 做 **状态驱动 (state-based)** 参数集 (站、跑、蹲、瞄准) 是非常重要的一步，使武器摆动在不同战况下显得差异化和有“手感层次”。
* \[ ] 引入 **自然噪声 (Perlin / Lissajous)** 用以 idle 微抖，会让武器看起来更“活”，玩家感觉手持感 +真实感强。
* \[ ] 参数调节

#### 代码

```cpp
// Utility: RotatorSpringInterp (实现与 VectorSpringInterp 类似)
FRotator RotatorSpringInterp(const FRotator& Current, const FRotator& Target, FRotatorSpringState& State, 
                             float Stiffness, float Damping, float DeltaTime, float Mass)
{
    // 这里给出简化数值积分（显式欧拉或 semi-implicit），在工程中你可以用更稳健的求解器
    FVector cur = FVector(Current.Pitch, Current.Yaw, Current.Roll);
    FVector targ = FVector(Target.Pitch, Target.Yaw, Target.Roll);

    // velocity stored in State (as FVector)
    FVector acc = (targ - cur) * Stiffness - State.Velocity * Damping;
    acc = acc / Mass;
    State.Velocity += acc * DeltaTime;
    FVector next = cur + State.Velocity * DeltaTime;
    return FRotator(next.X, next.Y, next.Z);
}

// 改良 WeaponSway_Look
void UUCharacterAnimHelpComp::WeaponSway_Look(
    FVector& swayOffset, FRotator& swayRot, FVectorSpringState& PosSpringState, FRotatorSpringState& RotSpringState,
    FRotator& lastControllerRot, float deltaSeconds, const FSwayParams& Params, ESwayState CurrentState,
    float CurrentTime, bool bHasInput)
{
    if (deltaSeconds <= KINDA_SMALL_NUMBER) return;

    // 1) 获取控制旋转并计算 delta
    FRotator currentControl = GetPawn<APawn>()->GetControlRotation();
    FRotator deltaRot = UKismetMathLibrary::NormalizedDeltaRotator(currentControl, lastControllerRot);
    lastControllerRot = currentControl;

    // 2) 角速度 (deg/s) and accel
    FVector2D rotSpeedFrame(deltaRot.Yaw / deltaSeconds, deltaRot.Pitch / deltaSeconds); // yaw, pitch
    // 用低通滤波平滑 rotSpeed（成员变量：PrevRotSpeedFiltered）
    const float alpha = FMath::Clamp( deltaSeconds * 10.f, 0.0f, 1.0f ); // time constant
    RotSpeedFiltered = FMath::Lerp(RotSpeedFiltered, rotSpeedFrame, alpha); // FVector2D stored
    FVector2D rotAccel = (RotSpeedFiltered - PrevRotSpeedFiltered) / deltaSeconds;
    PrevRotSpeedFiltered = RotSpeedFiltered;

    // 3) 归一化速度（参考 fiducial）
    FVector2D rotSpeedNorm = RotSpeedFiltered / Params.FiducialRotSpeed;
    rotSpeedNorm.X = FMath::Clamp(rotSpeedNorm.X, -1.f, 1.f);
    rotSpeedNorm.Y = FMath::Clamp(rotSpeedNorm.Y, -1.f, 1.f);

    // 4) 生成目标偏移（translation）——可以用非线性映射，增加高速度响应
    // 使用曲线: sign * pow(abs(norm), 1.2) 可以让高转速更敏感
    auto nonLinear = [](float v){ return FMath::Sign(v) * FMath::Pow(FMath::Abs(v), 1.2f); };
    float nx = nonLinear(rotSpeedNorm.X), ny = nonLinear(rotSpeedNorm.Y);

    FVector swayOffsetTarget = FVector(
        nx * Params.SwayOffsetMulti.X * -1.f,       // yaw → lateral opposite
        nx * Params.SwayOffsetMulti.Y,              // yaw → forward/back small
        ny * Params.SwayOffsetMulti.Z               // pitch → vertical
    );

    // 5) 旋转目标（rotation）单独计算（不要从位置直接映射）
    FRotator rotTarget = FRotator(
        ny * Params.SwayRotMulti.Pitch * -1.f, // pitch
        nx * Params.SwayRotMulti.Yaw * -1.f,   // yaw (we'll later apply yaw on shoulder pivot)
        nx * Params.SwayRotMulti.Roll          // roll, responsive to yaw
    );

    // 6) 应用 Lead/Lag：对 target 做滞后（exponential）让武器滞后视角
    // 这里用简单的一阶低通滤波来实现 lag；当 LeadLag < 0 可变成带超前逻辑（更复杂）
    float lagFactor = FMath::Clamp(Params.LeadLag, -1.f, 1.f);
    if (lagFactor > 0.f) {
        // lag amount (0..1)
        float tau = FMath::Lerp(0.02f, 0.5f, lagFactor); // time constant
        float k = deltaSeconds / (tau + deltaSeconds);
        TargetOffsetFiltered = FMath::Lerp(TargetOffsetFiltered, swayOffsetTarget, k);
        TargetRotFiltered = FMath::Lerp(TargetRotFiltered, FVector(rotTarget.Pitch, rotTarget.Yaw, rotTarget.Roll), k);
        swayOffsetTarget = TargetOffsetFiltered;
        rotTarget = FRotator(TargetRotFiltered.X, TargetRotFiltered.Y, TargetRotFiltered.Z);
    } else if (lagFactor < 0.f) {
        // light lead: apply small predictive term using rotAccel
        float leadAmp = FMath::Abs(lagFactor);
        swayOffsetTarget += FVector(rotAccel.X, 0.f, rotAccel.Y) * Params.SwayOffsetMulti * leadAmp * 0.05f;
        rotTarget.Pitch += rotAccel.Y * Params.SwayRotMulti.Pitch * leadAmp * 0.02f;
    }

    // 7) 在没有输入时处理 idle noise & state ramp
    float stateRamp = ComputeStateRamp(CurrentState, CurrentTime, Params); // returns 0..1 according to delay/rampTime
    if (!bHasInput && rotSpeedNorm.SizeSquared() < 0.01f) {
        // Idle noise — 小幅 Perlin 或 Lissajous
        FVector noise = GetIdleNoise(CurrentTime) * 0.01f; // e.g., returns ~[-1,1]
        swayOffsetTarget += noise * Params.SwayOffsetMulti * 0.3f * stateRamp;
        // small rotational noise:
        rotTarget += FRotator(noise.Z * Params.SwayRotMulti.Pitch * 0.2f, noise.X * Params.SwayRotMulti.Yaw * 0.15f, noise.Y * Params.SwayRotMulti.Roll * 0.25f);
    }

    swayOffsetTarget *= stateRamp;
    rotTarget = rotTarget * stateRamp;

    // 8) 弹簧插值（位置）
    swayOffset = UKismetMathLibrary::VectorSpringInterp(swayOffset, swayOffsetTarget, PosSpringState, Params.Pos_Stiffness, Params.Pos_Damping, deltaSeconds, Params.Pos_Mass);

    // 9) 旋转弹簧插值
    swayRot = RotatorSpringInterp(swayRot, rotTarget, RotSpringState, Params.Rot_Stiffness, Params.Rot_Damping, deltaSeconds, Params.Rot_Mass);

    // 10) 限幅
    if (swayOffset.Size() > Params.MaxOffset) {
        swayOffset = swayOffset.GetSafeNormal() * Params.MaxOffset;
    }
    swayRot.Pitch = FMath::Clamp(swayRot.Pitch, -Params.MaxPitch, Params.MaxPitch);
    swayRot.Yaw   = FMath::Clamp(swayRot.Yaw,   -Params.MaxYaw,   Params.MaxYaw);
    swayRot.Roll  = FMath::Clamp(swayRot.Roll,  -Params.MaxRoll,  Params.MaxRoll);

    // 11) 注意：Yaw 的最终应用应在 AnimBP 层绕 ShoulderPivot（见下一节）
}

```

## 被抛弃的方案

### Spring Arm

> 几乎不可用，但在大量初学者教程中出现

勾选Enable Camera Rotation Lag，使用全身延迟旋转\
[Weapon Sway in UE4- THE CORRECT WAY | Black Ops Zombies in UE4 #6 - YouTube](https://youtu.be/rZUfkXaVvX4)\
缺点：不可控，无法限制摆动幅度，并不能真正解决问题\
Todo: 询问GPT用视频帧分析摄像机抖动要怎么做

# weapon bobbing

### 基本概念：

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

# Shoot Sway

[How I Made My FPS Game Feel Better To Play](https://www.youtube.com/watch?v=SWf9pNdivpo\&t=8s)\
视频中一开始采用随机晃动让人眩晕，后来采用purlin noise获得平滑的摄像机抖动![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/PixPin_2025-11-28_11-02-18.gif)

[自制COD20 优化开火动画](https://www.bilibili.com/video/BV1dRffYjEra)

# 移动摇晃

[Realistic Headbob Camera Shakes](https://www.youtube.com/watch?v=lwM-SbDKnxQ)

# Idle
