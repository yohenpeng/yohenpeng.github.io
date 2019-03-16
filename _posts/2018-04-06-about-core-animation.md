---
layout: post
title: 核心动画读书笔记(一)
description: 核心动画读书笔记
category: blog
---


##  图层树
*Core Animation听起来是很容易让人误以为这是个动画框架，类似于Core Image,Core Foundation的内置框架，但是其实动画只是Core Anmiation特性的一部分，更多的是一个复合引擎，它的职责就是尽可能快的组合屏幕上不同的可视内容，分解成独立的图层，存储在一个叫做图层树的体系中，于是这个树形成了UIKit以及iOS应用程序之中你能在屏幕上看见的一切基础。*

#### UIView & CALayer
每一个UIview都有一个CALayer实例的图层属性，也就是所谓的backing layer，视图的职责就是创建并管理这个图层，以确保当子视图在层级关系中添加或者被移除的时候，他们关联的图层也同样对应在层级关系树当中有相同的操作。实际上这些UIView背后关联的图层才是真正用来在屏幕上显示和做动画的，UIView只是对它的一个封装。封装的抽象性越高，也就越不灵活，类似于以下功能就没有暴露出来。
1. 阴影，圆角，带颜色的边框
2. 3D 变换
3. 非矩形范围
4. 透明遮罩
5. 多级非线性动画

所以我们有必要去探索Core Animation的一切。

为什么iOS要基于UIView和CALayer提供两个平行的层级关系呢？为什么不用一个简单的层级来处理所有事情呢？原因在于要做职责分离，这样也能避免很多重复代码

## 寄宿图
#### contetns属性
###### contents 
寄宿图，一般情况下都是用来赋值，将UIImage转换成CGImage，之所以contents类型是id，是因为在Mac OS系统上CGImage和NSIamge类型的值都有效果。

(那平时我们没有设置寄宿图的Layer，只是设置一些属性类似backgroundColor，在屏幕上渲染之后，我们可以直接访问吗？答案是不可以，对于一个 UIView，如果不提供 bitmap （设置 layer 的 contents 属性），也不实现 drawRect 或者 CALayer drawing 的那几个 draw 函数，那么一个 UIView 的内容会根据这个 UIView 的状态（ backgroundColor，bounds，其实都是 layer 的状态），提交到 render server，并由 render server 调用底层的图形 API 来绘制这个 UIView 的内容，并将最终绘制的 bitmap 结果通过类似 glReadPixels 的 API 从显寸上复制到 CPU 内存里，这部分内存由 CALayer 的内部私有变量来负责持有，这部分数据目前是无法直接获取到的，只能通过 CALayer 的 renderInContext:等 API 间接的取到，这里与 CALayer 的 contents 属性不同，contents 只是用来为一个 CALayer 提供内容的。 
所以一个 UIView 有三种方式来生成内容，分别是 drawRect：，设置 layer 的 contents 属性，设置 UIView 本身或者 layer 的属性，这三种方式最终内存占用是一模一样的，但是消耗的 CPU 和 GPU 资源却不同，第一种需要调用 CPU 绘图 API 来绘制图形，CPU bound ；第二种虽然不需要调用 CPU 来绘图，但是可能会使用 CPU 来解压缩图片文件，所以也是 CPU bound ；第三种 CPU 只负责必要的状态修改，由 GPU 来完成主要的绘图工作，因为 GPU 绘图效率比 CPU 高，所以这种方式应该是效率最高的，整体的性能消耗也最低，GPU bound)


###### contentsGravity
和UIImageView的contentMode类似，目的是为了决定内容在图层的边界中怎么对齐。其中:
kCAGravityResize等价于UIViewContentModeScaleToFill，
kCAGravityResizeAspect等价于UIViewContentModeScaleAspectFit，
kCAGravityResizeAspectFill等价于UIViewContentModeScaleAspectFill

###### contentsScale
定义了寄宿图的像素尺寸和视图大小的比例(sizeof(image)/sizeof(view))，默认情况下他是一个值为1.0的浮点数 ,将回以每个点1个像素绘制图片，如果设置为2.0，则会以每个点2个像素绘制图片，这就是retina屏幕。一般情况下可如下设置
```objectivec
layer.contentsScale = [UIScreen mainScreen].scale
```

###### maskToBounds
在View层有个叫clipsToBounds的属性可以相对应，默认情况下，UIView仍然会绘制超过边界的内容或者子视图，在CALayer下也是这样。如果设置maskToBounds为YES则会修剪超过边界的内容。

(使用CALayerDelegate绘制寄宿图不支持绘制超过边界的内容，使用CAShapeLayer支持绘制超过边界的内容)

###### contentsRect
这个属性允许我们在图层边框内显示寄宿图的一部分，需要注意的是这里使用单位坐标，0到1之间。比如我们取左上角部分：
```
UIImage *image = [UIImage imageNamed:@"test"];
sublayer.contents = (__bridge id _Nullable)(image.CGImage);
sublayer.contentsRect = CGRectMake(0, 0, 0.5, 0.5);
```

###### contentsCenter
可拉伸图片，这里也是使用单位坐标，0到1之间，和UIImage的resizableImageWithCapInsets类似。


#### Custom Drawing
如果不需要自定义绘图都能实现需求，那就不要创建这个方法，如果实现了drawRect:方法，系统就会为视图分配一个寄宿图，这个寄宿图的像素尺寸等于视图大小乘以contentScale的值，这会造成CPU资源和内存的浪费，这也是为什么苹果建议，如果没有自定义绘制的任务就不要在子类中写一个空的drawRect方法。
当一个Layer需要绘制的时候，会请求他的代理给他一个寄宿图来显示
```
-(void)displayPayer:(CALayer *)layer;
```

如果代理没有这个方法，CAlayer就会转而尝试调用下面这个方法：
```
-(void)drawLayer:(CALayer *)layer inContext:(CGContextRef)ctx;
```
调用这个方法之前，会创建一个Core Graphics的绘制上下文，作为ctx参数传入。
现在回想UIView实现的drawRect里面获取到的绘制上下文就是这个时候创建的。
```
-(void)drawRect:(CGRect)rect{
    ...
    CGContextRef ctx = UIGraphicsGetCurrentContext();
    ...
}
```

## 图层几何学
#### 布局
* frame 是相对于父视图的位置坐标，而且bounds对应的是自己内部坐标系;
* frame 其实是个虚拟属性，根据bounds，center，transform合成而来，并不是独立存在；

#### 锚点
* anchorPoint 是用来移动图片的把柄，默认位于图层的中心点；
* View的center/Layer的position 都是anchorPoint相对于父试图的位置；
* position和anchorPoint修改其中任何一个值，都不会影响另外一个。anchorPoint是position在图层内部的相对位置；

#### 坐标系
zPosition可以用来改变图层的显示顺序，越大就在上面，但不可以改变事件传递的顺序；
坐标系之间转换method：
```
- (CGPoint)convertPoint:(CGPoint)point toView:(nullable UIView *)view;
- (CGPoint)convertPoint:(CGPoint)point fromView:(nullable UIView *)view;
- (CGRect)convertRect:(CGRect)rect toView:(nullable UIView *)view;
- (CGRect)convertRect:(CGRect)rect fromView:(nullable UIView *)view;
```

#### Hit Testing
View和Layer方法都能找到对应的，我们都可以实现下面方法来改变响应者选择的逻辑
```
//UIView
- (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;
- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;

//Layer
- (nullable CALayer *)hitTest:(CGPoint)p;
- (BOOL)containsPoint:(CGPoint)p;
```

#### 自动布局
* frame绝对布局，代码效率低，性能好
* autoLayout，代码效率高，随着视图层次增加，性能较差
* 第三方布局库，方案还是挺多的，比如facebook的YogaKit，ComponentKit


## 视觉效果
#### 圆角
* cornerRadius 这个曲率值只影响背景颜色而不影响背景图片或是子图层
* masksToBounds 设置为YES的话，图层里面的所有东西都会被截取


#### 图层边框
* borderWidth 边框宽度
* borderColor 边框颜色

#### 阴影
* shadowOpacity 不透明度 0.0（不可以见）～1.0（完成不透明）
* shadowColor   默认是黑色
* shadowOffset  阴影的方向和距离，默认值是CGSize（0，-3）
* shadowRadius  阴影的模糊度，越大阴影越模糊，图层看上去就会更明显

###### shadowPath属性
我们已经知道图层阴影并不是总是方的，而是从图层内容的形状继承而来。但是实时计算阴影是一个非常消耗资源的，特别是很多子图层，每个图层还有一个有透明效果的寄宿图的时候。如果事先知道阴影的样子，你可以指定一个shadowPath来提高性能

```
[myView.layer setShadowPath：[[UIBezierPath bezierPathWithRect：myView.bounds] CGPath];
```
#### 图层蒙板
mask图层定义了父图层的部分可见区域，mask图层的Color属性是无关紧要的，真正重要的是图层的轮廓。
CALayer蒙板图片真正厉害的地方在于不局限于静态图，可以代码实时生成。

#### 拉伸过滤
当素材的尺寸和图层大小比例刚好1:1是最好的，像素没有被压缩也没有被拉伸，CPU不需要为此额外的计算

* KCAFilterLinear 双线性滤波算法
* kCAFilterNearest 没有斜线的小图应该用最近过滤
* KCAFilterTrilinear  三线性滤波算法，存储了多个大小情况下的图片，也叫多重贴图

#### 组透明
组透明其实用光栅化实现，如果shouldRasterize设置为YES，在应用透明度之前，其子图层都会整合成一个整体的图片，这样就没有透明度混合的问题了。
```
button.layer.shouldRasterize = YES;
button.layer.rasterizationScale = [UIScreen mainScreen].scale;
```

## 变换
旋转角度使用左手坐标系，拇指为X Y Z方向。
```
self.view.layer.affineTransform = CGAffineTransformMakeScale(2, 2);
self.view.layer.transform = CATransform3DMakeScale(2, 2, 2);
```

3D转换：由于尽管Core Animation图层存在于3D空间之中，但他们并存在同一个3D空间，每个图层的3D场景其实都是扁平化的，当你从正面观察一个图层，看到的实际上由子图层创建的想象出来的3D场景，但当你倾斜这个图层，你会发现实际上这个3D场景仅仅是被绘制在图层的表面。

固体对象这些日常使用较少，后面再补充。

## 专用图层
* CAShapeLayer 通过矢量绘制图形而不是bitMap绘制的图层子类，直接将数据提交到GPU来处理
* CATextLayer  其实iOS7之后，使用textLayer和直接使用富文本性能上没傻差别，因为现在UILabel使用的TextKit，其实也是基于CoreText封装的
* CATransformLayer  
* CAGradientLayer 颜色渐变图层，使用硬件加速
* CAReplicationLayer 高效生成许多类似的图层
* CAScrollLayer 
* CATiledLayer  适合处理大图
* CAEmittedLayer 高性能粒子引擎
* CAEAGLLyaer  OPENGL图层
* AVPlayerLayer  播发视频

## 隐式动画
#### 事务
没有关联View层的Layer，改变CAlayer的一个可做动画的属性，都会有默认动画，持续时间0.25s,次数为1次。这一切都是默认的行为，你不需要做额外的操作。

即使你没有显示用[CATransaction begin]和[CATransaction commit]开始和提交事务，任何一次在run loop循环中属性的改变都会被集中起来，然后做一次0.25秒的动画。

其实UIView中的beginAnimation:context:和commitAnimations:context:都是对应上面事务的操作。

#### 图层行为
对于有关联View的Layer，我们称之为rootLayer，默认把隐式动画都关闭了，下面的流程第一步就返回了nil。
1. 检查CALayerDelegate的actionForLayer：forKey，有直接调用并返回结果；
2. 没有委托或者委托没有实现，则检查自身的actions字典；
3. 如果actions字典没有包含对应的属性，那么图层接着在它的style字典中搜索属性名；
4. 如果在style也找不到对应的行为，那么图层将会直接调用定义每个属性的标准行为的defaultActionForKey:方法。

过渡动画
```
CATransition *transition = [CATransition animation];
transition.tyle = kCATransitionPush;
transition.subtyle = KCATransitionFromLeft;
self.colorLayer.actions = @{@"backgroundColor":transition};
```

模型树/呈现树/渲染树，其中渲染树实在SpringBoard进程里面的，我们无法获取。呈现树就是我们屏幕中看到的状态，呈现树和模型树可以相互转换.

## 显示动画
#### 属性动画
想在动画完成后，同步更新Layer的属性值，我们可以在animationDidStop:finished:中操作，假如是非rootLayer，我们需要设置一个新的事务，并且禁用图层行为，否则动画会发生两次，一次是之前的显式动画，另一次是因为结束之后的隐式动画。
```
[CATransaction begin];
[CATransaction setDisableAction:YES];
//更新属性值...
[CATransaction commit];
```

为了在动画结束回调中能区分动画对象，我们可以对animation进行KVC操作
```
[baseAnimation setValue:self.animationView forKey:@"handleView"];
```
###### 关键帧动画
CAKeyframeAnimation 关键帧动画，除了提供一个数组的值可以按照颜色变化做动画，还有另外一种方式指定动画，就是使用CGPath。path属性可以用一种直观的方式，使用Core Graphics函数定义运行序列来绘制动画。
```
CAKeyframeAnimation *animation = [CAKeyframeAnimation animation];
animation.keyPath = @"position";
animation.duration = 4.0;
animation.path = bezierPath.CGPath;
animation.rotationMode = KCAAnimationRotateAuto;
[shipLayer addAnimation:animation forKey:nil];
```
其中设置KCAAnimationRotateAuto能够使图层根据曲线的切线自动旋转。

###### 虚拟属性
对transfrom属性做动画经常使不符合预期的，这是因为变换矩阵不能和其他类似position.x的属性值做单纯的数值加减。

下面这些使苹果提供给我们做动画使用的keyPath，其中toValue和byValue需要区分，绝对值和相对值。
* transform.rotation.x 围绕x轴翻转 参数：角度 angle2Radian(4) 
* transform.rotation.y 围绕y轴翻转 参数：同上 
* transform.rotation.z 围绕z轴翻转 参数：同上 
* transform.rotation 默认围绕z轴 
* transform.scale.x x方向缩放 参数：缩放比例 1.5 
* transform.scale.y y方向缩放 参数：同上 
* transform.scale.z z方向缩放 参数：同上 
* transform.scale 所有方向缩放 参数：同上 
* transform.translation.x x方向移动 参数：x轴上的坐标 100 
* transform.translation.y x方向移动 参数：y轴上的坐标 
* transform.translation.z x方向移动 参数：z轴上的坐标 
* transform.translation 移动 参数：移动到的点 （100，100） 
* opacity 透明度 参数：透明度 0.5 
* backgroundColor 背景颜色 参数：颜色 (id)[[UIColor redColor] CGColor] 
* cornerRadius 圆角 参数：圆角半径 5 
* borderWidth 边框宽度 参数：边框宽度 5 
* bounds 大小 参数：CGRect 
* contents 内容 参数：CGImage 
* contentsRect 可视内容 参数：CGRect 值是0～1之间的小数 
* hidden 是否隐藏 
* position 
* shadowColor 
* shadowOffset 
* shadowOpacity 
* shadowRadius

###### 动画组
对CAAnimationGroup.animations操作即可。


###### 过渡
属性动画只对图层的可动画属性起作用，所以如果要改变一个不能动画的属性（比如图片），或者从层级关系中添加或者移除图层，属性动画将不起作用，于是有了过渡的概念。过渡并不像属性动画那样平滑地在两个值之间做动画，而是影响到整个图层的变化。过渡动画首先展示之前的图层外观，然后通过一个交换过渡到新的外观。
CATransition的type:
1. kCATransitionFade 淡入淡出效果
2. kCATransitonMoveIn 新的图层外观进来
3. kCATransitionPush 2和4的效果叠加
4. kCATransitionReveal 旧的图层外观出去

subType:
1. kCATransitonFromRight
2. kCATransitonFromLeft
3. kCATransitonFromTop
4. kCATransitonFromBottom

和属性动画不同，对指定的图层一次只能使用一次CATransition，因此，无论你对动画的键设置什么值，过渡动画都会对它的键设置为“transition”，也就是常量kCATransition。

对图层树的动画：
要确保CATransition添加到的图层在过渡动画发生时不会在树状结构中被移除，否则CATransition将会和图层一起被移除，这里可以将动画添加到被影响图层的superLayer。

###### 在动画过程中国取消动画
```
-(void)removeAnimationForKey:(NSString*)key;
-(void)removeAllAnimations;
```

待续....

