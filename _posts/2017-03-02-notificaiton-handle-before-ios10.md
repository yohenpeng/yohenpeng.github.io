---
layout: post
title:  iOS 10和低版本下，前后台收到消息处理方案
description: iOS 10和低版本下，前后台收到消息处理方案
category: blog
---


由于iOS 9及以下版本，前台收到通知时无法显示在通知栏的。iOS 10 已经开放了前台展示通知栏的API。

首先我们来看看低版本的如何处理：
```
-(void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo;
-(void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler;
```
首先来比较这两个API的异同，虽然前者已经被苹果抛弃了，但是在低版本系统我们还是要适配的，最主要的区别是前者只能在应用跑在前台时才能收到，后者则前后台都可以收到，而且如果设置了后台模式为Remote Notifications的话，还可以执行30s来获取数据。

假如两者都在Appdelegate里面都实现的话，系统只会调用带completionHandler的后者。
为了前后台通知处理一致，我们实现后者，大致如下：
```
-(void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult))completionHandler{
     //如果是前台，使用第三方EBForeNotification定制通知栏界面，假如在后台或者未运行，则本来就有
    if ([UIApplication sharedApplication].applicationState == UIApplicationStateActive) {
        [EBForeNotification handleRemoteNotification:userInfo soundID:0 isIos10:NO];
    }
     completionHandler(UIBackgroundFetchResultNoData);
}
```

下面处理iOS 10的情况：
new API 设置前台收到远程消息时是否显示
```
-(void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler{
    
    completionHandler(UNNotificationPresentationOptionAlert);
}
```
用户点击通知栏，前后台处理方式一致，需要注意的是以前的低版本的API是收到通知就回调，iOS 10以后则是用户点击才回调
```
-(void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)())completionHandler{
        //do something
}
```
