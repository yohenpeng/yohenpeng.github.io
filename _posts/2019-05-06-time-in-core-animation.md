---
layout: post
title: CoreAnimation时间坐标系理解
subTitle: 你看到的一切图像都是时间的流逝造就的
description: 核心动画读书笔记
catalog: true
tags:
    - iOS
    - 动画时间
---

## 时间概念
首先来看一个时间函数CACurrentMediaTime(),Apple的官方说明是：
*Returns the current CoreAnimation absolute time. This is the result of calling mach_absolute_time () and converting the units to seconds.*

返回的是当前CoreAnimation坐标系的绝对时间，自系统启动以来的总时间（剔除系统休眠的时间），所有图层时间都是参照CACurrentMediaTime()，由于控制动画的开始时间或者速度等原因，而且图层之间的时间还可能不一致的。

系统提供了两个API给到我们用于转换时间
```
- (CFTimeInterval)convertTime:(CFTimeInterval)t fromLayer:(nullable CALayer *)l;
- (CFTimeInterval)convertTime:(CFTimeInterval)t toLayer:(nullable CALayer *)l;
```

其中layer当前时间t这可以用下面这个表达式计算,其中第二个参数我们传nil。
```
[layer convert:CACurrentMediaTime() fromLayer:nil]
```

## begin、speed、timeOffset对t的影响
这里就不去猜begin、speed、timeOffset和图层时间t的关系了，直接去到CAMediaTiming.h头文件找到他们的关系：
```
t = (tp - begin) * speed + timeOffset;
```

下面我们通过实验来验证来验证三者对t的影响
1、beginTime（晚多少秒开始，受speed的影响）

self.view.layer.beginTime = 0;
self.layer_red.beginTime = 1;
NSLog(@"tp's time:%0.2f",[self.view.layer convertTime:CACurrentMediaTime() fromLayer:nil]);
NSLog(@"t's time:%0.2f",[self.layer_red convertTime:CACurrentMediaTime() fromLayer:nil]);

不改speed的情况下，我们可以看到后者要晚上1秒钟，如果self.view.layer和self.layer_red同时加入一个动画，那么后者就会晚1秒才才开始播放


2、timeOffset (时间偏移量，不受speed的影响)

self.view.layer.beginTime = 0;
self.layer_red.beginTime = 0;
self.layer_red.timeOffset = 1;
NSLog(@"tp's time:%0.2f",[self.view.layer convertTime:CACurrentMediaTime() fromLayer:nil]);
NSLog(@"t's time:%0.2f",[self.layer_red convertTime:CACurrentMediaTime() fromLayer:nil]);

我们看到后者比前者快1秒钟，如果加入动画的话，后者则直接从1秒处播放，前面1s我们都看不见


3、speed（时间流逝的速度）
self.layer_red.beginTime = 0;
self.layer_red.timeOffset = 0;
self.layer_red.speed = 2;

dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    NSLog(@"tp's time:%0.2f",[self.view.layer convertTime:CACurrentMediaTime() fromLayer:nil]);
    NSLog(@"t's time:%0.2f",[self.layer_red convertTime:CACurrentMediaTime() fromLayer:nil]);
});

我们到过了2s，后者的时间就过了4s，比前后快了2s，看来就像快进了一倍。

## begin、speed、timeOffset之间的关系
一个layer初始状态下，而且superLayer没有改变begin、speed、timeOffset情况下，本地图层时间t就是CACurrentMediaTime()，因为根据公式没有改变属性的layer的speed为1，begin和timeOffset为0，
```
t = (tp - 0) * 1 + 0 = tp = CACurrentMediaTime();
```

下面的代码输出可以验证上面表达式
```
[self.view.layer addSublayer:self.layer_red];
self.layer_red.frame = CGRectMake(100, 100, 50, 50);
NSLog(@"superlayer t:%0.2f",[self.view.layer convertTime:CACurrentMediaTime() fromLayer:nil]);
NSLog(@"CACurrentMediaTime():%0.2f",CACurrentMediaTime());
NSLog(@"layer t :%0.2f",[self.layer_red convertTime:CACurrentMediaTime() fromLayer:nil]);
```

## 动画暂停暂停的和恢复
暂停的话，也就是本地时间要静止，speed为0的本地图层时间才不会继续往前走，我们可以拿到当前图片本地图层时间,赋值给timeOffset就可以保证图层本地时间不变；
```
CFTimeInterval t = [self.layer_red convertTime:CACurrentMediaTime() fromLayer:nil];
self.layer_red.speed = 0;
self.layer_red.timeOffset = t;
```
    
继续的话，speed要从0恢复出来，但是本地图层时间要晚暂停这段时间段大小，我们知道要改变begin时间
```
CFTimeInterval pauseTime = self.layer_red.timeOffset;
self.layer_red.beginTime = 0;
self.layer_red.timeOffset = 0;
self.layer_red.speed = 0;
CFTimeInterval curentTime = [self.layer_red convertTime:CACurrentMediaTime() fromLayer:nil];
CFTimeInterval delayTime = curentTime - pauseTime;
self.layer_red.beginTime = delayTime;
```



