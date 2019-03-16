---
layout: post
title:  自定义TableCell删除效果手势问题解决
description: 自定义TableCell删除效果手势问题解决
category: blog
---

今天尝试给UITableViewCell增加自定义左滑手势，发现ContentView的frame是无法修改的，只能在上面增加一层MyContentView，并加上一个pan手势，当往左开始移动的时候，然后在MyContentView下插入一个UIButton。然后效果出来了，但是，但是发现UITableView上下方向无法滑动了。

看来是添加在MyContentView的手势和UITableView自带的手势冲突了
看了下代理方法，应该是UITableView的手势被阻止了，可以通过下面的方法return YES来启用。
```
- (BOOL)gestureRecognizer:(UIGestureRecognizer*)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer*)otherGestureRecognizer;
```

现在好了，现在可以上下滑动和左拉滑动了，但是手势会同时响应，这个太bug了，看到网上的做法，可以通过获得手势在某一个方向上的速度来判定：

```
-(BOOL)gestureRecognizerShouldBegin:(UIPanGestureRecognizer *)gestureRecognizer{

    if([gestureRecognizer isKindOfClass:[UIPanGestureRecognizer class]]){
        CGPoint velocityPoint =[gestureRecognizer velocityInView:self.contentView];
        if(fabs(velocityPoint.x)> 100)return YES;
        else return NO;
    }else{
        return NO;
    }

}
```

顺便回顾了iOS的事件传递和 总结下，手势具有优先权。给某个视图加上手势，当有触摸事件发生的时候，视图的touchBegan还是会执行的，如果触摸过程被捕获为手势事件，就不会回到touchMove等方法了，假如不能被识别为手势事件，就跟往常一样会进行各种touch方法的回调。